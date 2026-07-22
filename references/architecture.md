# Arquitetura: Clean Architecture, Vertical Slice e DDD/CQRS

## Duas formas de organizar o código — e por que a resposta certa costuma ser "as duas"

Existem dois eixos de organização que resolvem problemas diferentes, e é comum confundi-los como se fossem concorrentes:

- **Clean Architecture** organiza o código **por camada técnica** (Domain, Application, Infrastructure, Api). Sua força é a regra de dependência: protege o domínio de vazar detalhes de infraestrutura, e permite trocar EF Core por outra coisa sem tocar na regra de negócio.
- **Vertical Slice Architecture** organiza o código **por feature/caso de uso** (`CreateOrder`, `SubmitOrder`, `GetOrderById`), cada slice contendo tudo que aquela funcionalidade precisa. Sua força é localidade: para entender ou mudar "criar pedido", você abre uma pasta, não pula entre quatro projetos.

Elas não competem — resolvem problemas ortogonais (organização por *o que* o código faz vs. por *que tipo* de código é). A recomendação padrão desta skill é a combinação das duas (ver seção dedicada abaixo), mas primeiro entenda cada uma isoladamente.

Para um CRUD simples (poucas entidades, sem regra de negócio real além de validação de campo), qualquer uma dessas estruturas isolada já é overhead — uma Minimal API direta com um serviço de aplicação simples e EF Core é a escolha certa. Ao propor arquitetura, sempre explique a troca em vez de aplicar um padrão por reflexo.

## Opção 1: Clean Architecture (por camada)

```
src/
├── MyApp.Domain/           # Entidades, Value Objects, Domain Events, interfaces de repositório
├── MyApp.Application/      # Casos de uso (Commands/Queries + Handlers), DTOs, validação
├── MyApp.Infrastructure/   # Implementação de repositórios, EF Core, integrações externas
└── MyApp.Api/               # Controllers/Minimal API endpoints, composição (DI), middlewares
```

Regra de dependência: as setas apontam para dentro. `Domain` não depende de nada. `Application` depende só de `Domain`. `Infrastructure` e `Api` dependem de `Application`/`Domain`, nunca o contrário. Se você perceber uma entidade de domínio referenciando `Microsoft.EntityFrameworkCore`, isso é um sinal de que a camada está vazando.

O custo dessa abordagem aparece quando o projeto cresce: implementar "criar pedido" de ponta a ponta exige tocar em quatro projetos diferentes (entidade no `Domain`, handler no `Application`, configuração EF no `Infrastructure`, endpoint no `Api`). Cada camada também tende a acumular código de features completamente não relacionadas lado a lado — o `Application` de um sistema grande vira uma pasta com centenas de handlers de dezenas de features misturados.

## Opção 2: Vertical Slice Architecture (por feature)

Vertical Slice organiza o código por caso de uso, não por tipo técnico. Cada slice é uma pasta autocontida com o Command/Query, o Handler, o Validator e (se for específico daquela feature) o endpoint:

```
src/
└── MyApp.Api/
    └── Features/
        └── Orders/
            ├── CreateOrder/
            │   ├── CreateOrderCommand.cs      # request + handler + endpoint, no mesmo arquivo ou pasta
            │   ├── CreateOrderValidator.cs
            │   └── CreateOrderEndpoint.cs
            ├── SubmitOrder/
            │   ├── SubmitOrderCommand.cs
            │   └── SubmitOrderEndpoint.cs
            └── GetOrderById/
                ├── GetOrderByIdQuery.cs
                └── GetOrderByIdEndpoint.cs
```

```csharp
// Features/Orders/SubmitOrder/SubmitOrderCommand.cs — tudo que essa feature precisa, num lugar só
public sealed record SubmitOrderCommand(Guid OrderId) : IRequest<Result<Unit>>;

public sealed class SubmitOrderHandler(IOrderRepository repository, IUnitOfWork unitOfWork)
    : IRequestHandler<SubmitOrderCommand, Result<Unit>>
{
    public async Task<Result<Unit>> Handle(SubmitOrderCommand request, CancellationToken ct)
    {
        var order = await repository.GetByIdAsync(new OrderId(request.OrderId), ct);
        if (order is null) return Result<Unit>.Failure("Pedido não encontrado.");

        order.Submit();
        await unitOfWork.SaveChangesAsync(ct);
        return Result<Unit>.Success(Unit.Value);
    }
}

public static class SubmitOrderEndpoint
{
    public static void Map(IEndpointRouteBuilder app) =>
        app.MapPost("/orders/{orderId:guid}/submit", async (Guid orderId, ISender mediator, CancellationToken ct) =>
        {
            var result = await mediator.Send(new SubmitOrderCommand(orderId), ct);
            return result.IsSuccess ? Results.NoContent() : Results.Problem(result.Error);
        });
}
```

Ganho: para entender ou mudar "submeter pedido", você abre uma pasta e vê o fluxo inteiro, sem pular entre projetos. Cada slice pode até tomar decisões técnicas diferentes quando fizer sentido (uma query de leitura pesada pode usar Dapper direto, enquanto o resto do sistema usa EF Core) sem que isso vaze para as outras features.

Risco a vigiar: sem nenhuma disciplina, Vertical Slice puro tende a duplicar lógica de domínio entre slices (duas features reimplementando a mesma regra de "pedido só pode ser submetido se tiver itens" cada uma à sua maneira) — é exatamente esse risco que a combinação abaixo resolve.

## Combinação ideal: Vertical Slice por fora, Clean Architecture por dentro

Na prática, a estrutura que mais escala bem em projetos .NET de produção não escolhe uma ou outra — usa Vertical Slice como a organização visível do código de aplicação, com um núcleo de domínio compartilhado organizado como Clean Architecture por baixo:

```
src/
├── MyApp.Domain/                     # compartilhado entre todas as slices — a única fonte de verdade das regras de negócio
│   ├── Orders/
│   │   ├── Order.cs                  # entidade, invariantes, métodos de negócio
│   │   ├── OrderLine.cs
│   │   └── IOrderRepository.cs       # interface, implementação vive na Infrastructure
│   └── Customers/
│       └── Customer.cs
│
├── MyApp.Infrastructure/             # compartilhado — implementação de persistência, integrações externas
│   └── Orders/
│       ├── OrderRepository.cs
│       └── OrderConfiguration.cs     # EF Core
│
└── MyApp.Api/
    └── Features/                     # uma pasta por feature — isso é o Vertical Slice
        └── Orders/
            ├── CreateOrder/          # Command + Handler + Validator + Endpoint desta feature
            ├── SubmitOrder/
            └── GetOrderById/         # Query que projeta direto pra DTO — nem toca no agregado Order
```

A regra prática que faz essa combinação funcionar: **toda regra de negócio (invariante, transição de estado válida) vive no `Domain`, nunca duplicada dentro de um handler de slice.** O handler de cada slice é fino — ele busca a entidade, chama um método de negócio nela (`order.Submit()`), e persiste. A "verticalidade" está na organização dos casos de uso (onde fica o Command, o Validator, o endpoint), não na duplicação da lógica de domínio.

Isso dá os dois ganhos ao mesmo tempo: você abre `Features/Orders/SubmitOrder/` e entende o caso de uso inteiro numa pasta (ganho do Vertical Slice), e ainda assim `Order.Submit()` é a única implementação da regra "não pode submeter pedido vazio" no sistema inteiro, reusada por qualquer slice, job em background, ou handler de mensageria que precise dela (ganho do Clean Architecture / DDD).

Quando propuser essa combinação para o usuário, deixe claro o tamanho do compromisso: ainda são pelo menos três projetos (`Domain`, `Infrastructure`, `Api`), então para um serviço muito pequeno com pouca regra de negócio, um Vertical Slice mais simples (sem separar `Domain`/`Infrastructure` em projetos próprios, só em pastas dentro de um único projeto) já entrega a maior parte do benefício de organização com bem menos cerimônia.

## Domain layer: entidades e value objects

Entidades têm identidade e ciclo de vida; mudanças de estado acontecem via métodos com significado de negócio, não setters públicos:

```csharp
public sealed class Order
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderLine> _lines = [];
    public IReadOnlyList<OrderLine> Lines => _lines;

    private Order(OrderId id, CustomerId customerId)
    {
        Id = id;
        CustomerId = customerId;
        Status = OrderStatus.Draft;
    }

    public static Order Create(CustomerId customerId) => new(OrderId.New(), customerId);

    public void AddLine(ProductId productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Só é possível adicionar itens a um pedido em rascunho.");

        _lines.Add(new OrderLine(productId, quantity, unitPrice));
    }

    public void Submit()
    {
        if (_lines.Count == 0)
            throw new InvalidOperationException("Não é possível enviar um pedido sem itens.");

        Status = OrderStatus.Submitted;
    }
}
```

Value Objects (ex.: `Money`, `Address`) não têm identidade — são comparados por valor. Modele-os como `record`/`record struct` (ver `code-style.md`).

## Domain Events

Use eventos de domínio quando uma mudança de estado precisa disparar efeitos colaterais em outras partes do sistema, sem acoplar a entidade a esses efeitos diretamente:

```csharp
public sealed record OrderSubmitted(OrderId OrderId, CustomerId CustomerId) : INotification;
```

A entidade registra o evento (numa lista interna); a camada de aplicação/infraestrutura publica o evento depois que a transação é confirmada (não antes — publicar antes do commit pode notificar sobre algo que ainda pode ser revertido).

## CQRS com MediatR

CQRS separa operações que mudam estado (Commands) de operações que só leem (Queries). Isso não exige bancos separados — a maioria dos sistemas usa o mesmo banco para os dois, mas se beneficia de handlers dedicados e simples, cada um fazendo uma coisa. Na prática, é o CQRS que torna a combinação Vertical Slice + Clean Architecture natural: cada Command/Query com seu Handler já É uma slice — não é preciso inventar outra unidade de organização:

```csharp
public sealed record SubmitOrderCommand(OrderId OrderId) : IRequest<Result<Unit>>;

public sealed class SubmitOrderHandler(IOrderRepository repository, IUnitOfWork unitOfWork)
    : IRequestHandler<SubmitOrderCommand, Result<Unit>>
{
    public async Task<Result<Unit>> Handle(SubmitOrderCommand request, CancellationToken ct)
    {
        var order = await repository.GetByIdAsync(request.OrderId, ct);
        if (order is null)
            return Result<Unit>.Failure("Pedido não encontrado.");

        order.Submit();
        await unitOfWork.SaveChangesAsync(ct);

        return Result<Unit>.Success(Unit.Value);
    }
}
```

Para leitura, prefira queries que retornam DTOs diretamente (via projeção EF Core ou Dapper), sem passar pela entidade de domínio — não há necessidade de reconstruir um agregado inteiro só para exibir uma lista. Veja `data-access.md`.

## Result pattern em vez de exceptions para fluxo esperado

Exceptions devem sinalizar o inesperado (bug, falha de infraestrutura). Para falhas de negócio esperadas ("pedido não encontrado", "saldo insuficiente"), prefira um tipo `Result<T>` que força quem chama a lidar com o caso de falha explicitamente, em vez de `try/catch` genérico:

```csharp
public readonly record struct Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    private Result(bool isSuccess, T? value, string? error)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }

    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
}
```

## Repository e Specification pattern

O repositório abstrai persistência atrás de uma interface definida no `Domain`, implementada no `Infrastructure`:

```csharp
// Domain
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    void Add(Order order);
}
```

Evite repositórios genéricos (`IRepository<T>`) com métodos como `GetAll()` sem filtro — eles tendem a vazar preocupações de query para fora da camada de dados e incentivam N+1 (ver `data-access.md`). Prefira métodos específicos por caso de uso, ou o Specification pattern quando os filtros forem realmente compostos dinamicamente.

## FluentValidation para regras de entrada

Separe validação de formato/entrada (FluentValidation, na borda da Application) de invariantes de domínio (dentro da entidade). A entidade nunca deve confiar que os dados já foram validados por fora — ela deve continuar sendo capaz de proteger seus próprios invariantes.

```csharp
public sealed class SubmitOrderCommandValidator : AbstractValidator<SubmitOrderCommand>
{
    public SubmitOrderCommandValidator()
    {
        RuleFor(x => x.OrderId.Value).NotEmpty();
    }
}
```
