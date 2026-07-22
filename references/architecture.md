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

### Fiação da slice: registro de endpoints e validators

Cada slice expõe um `Map(IEndpointRouteBuilder)` estático (como no exemplo acima). No `Program.cs`, MediatR e FluentValidation são registrados uma vez por varredura de assembly — nenhuma linha nova por feature, exceto a chamada de `Map`:

```csharp
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
// ...
CancelOrderEndpoint.Map(app);
SubmitOrderEndpoint.Map(app);
```

O validator roda como pipeline behavior do MediatR — todo Command passa por ele antes do handler, sem que cada endpoint precise chamar validação manualmente:

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        foreach (var validator in validators)
        {
            var result = await validator.ValidateAsync(request, ct);
            if (!result.IsValid)
                throw new ValidationException(result.Errors);
        }
        return await next();
    }
}
```

A `ValidationException` é convertida em `400 ProblemDetails` por um `IExceptionHandler` global registrado no `Program.cs` (`AddExceptionHandler` + `UseExceptionHandler`) — erro de formato de entrada nunca chega ao handler da slice.

## Projeto existente (brownfield): siga o layout consolidado

A combinação acima é a recomendação para código **novo** (greenfield). Em um codebase existente com outro layout consolidado — por exemplo, Clean Architecture clássica com `Application/Commands/`, `Application/Queries/` organizados por tipo técnico — **siga o layout existente ao adicionar features**. Não crie uma pasta `Features/` isolada no meio de um projeto organizado por tipo técnico: dois padrões concorrentes no mesmo repositório é pior que qualquer um dos dois sozinho.

Migrar um projeto por camadas para slices é uma decisão deliberada do time, feita incrementalmente e como trabalho próprio — nunca pegando carona na entrega de uma feature. Se o usuário sentir a dor que motiva a migração (Application com dezenas de handlers de features misturadas), proponha a migração como conversa separada, explicando custo e ganho.

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

### Transactional Outbox: quando o evento cruza a fronteira do processo

"Publicar depois do commit" resolve a ordem, mas não a atomicidade: banco e broker são duas escritas não atômicas (dual-write) — se o processo cai entre o commit e o publish, o evento se perde para sempre. Para eventos que **outro serviço** depende de receber (estorno, notificação, integração), use o Transactional Outbox:

1. **Gravar**: o handler serializa os eventos pendentes da entidade numa tabela `OutboxMessages` (`Id`, `Type`, `Payload` JSON, `OccurredAt`, `PublishedAt` nullable) — no **mesmo `SaveChangesAsync`** do agregado. Commit atômico: ou salva estado + evento, ou nada.
2. **Publicar**: um `BackgroundService` separado (ver `background-workers.md`) faz polling dos registros com `PublishedAt IS NULL`, publica no broker **com publisher confirms** (sem confirm, marcar como publicado é mentira), e só então preenche `PublishedAt`.
3. **Aceitar reentrega**: se o processo cai entre o publish e o marcar, a mensagem sai de novo no próximo ciclo — a garantia é *at-least-once*. O consumidor do outro lado precisa ser idempotente usando o `OutboxMessage.Id` como `MessageId` (obrigatório de qualquer forma — ver `background-workers.md`).

Para efeitos colaterais **dentro do mesmo processo** (ex.: handler MediatR de `INotification` atualizando uma projeção local), o outbox é overhead — publique in-process após o `SaveChanges`; se o processo cair, a transação inteira do caso de uso falhou junto e o retry do caso de uso refaz tudo. O outbox entra quando perder o evento deixa **outro sistema** inconsistente.

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

### Invariante de domínio vs falha de negócio esperada: como conectar os dois

A entidade protege invariantes lançando exception (`order.Submit()` lança se não há itens) — ela não conhece `Result<T>`, que é preocupação da aplicação. Quando a mesma condição é também uma falha **esperada** de um caso de uso (usuário tenta cancelar pedido já enviado via API), o handler faz a ponte de uma destas duas formas — escolha uma e seja consistente no projeto:

1. **Perguntar antes de agir** (preferida quando a regra é simples): a entidade expõe a pré-condição como leitura (`order.CanCancel`), o handler verifica e retorna `Result<T>.Failure(...)` sem disparar exception. O método `Cancel()` continua lançando — a exception vira a rede de segurança contra bug, não o fluxo normal.
2. **Converter no handler**: o handler chama o método de negócio dentro de `try/catch` da exception de domínio específica e converte em `Result<T>.Failure(ex.Message)`. Aceitável, mas usar exception como fluxo esperado tem custo de legibilidade e performance — prefira a forma 1 quando possível.

Nunca deixe a exception de invariante vazar até o cliente HTTP como `500` quando a condição é um erro de negócio esperado.

### Mapeando Result para status HTTP

`Error` como `string` única não distingue "não encontrado" de "regra de negócio violada" — e o endpoint precisa dessa distinção para devolver `404` vs `409`/`400`. Adicione uma categoria ao erro:

```csharp
public enum ErrorKind { NotFound, Conflict, Validation }

public sealed record Error(ErrorKind Kind, string Message);
// Result<T> passa a carregar Error em vez de string

// No endpoint da slice:
return result.IsSuccess
    ? Results.NoContent()
    : result.Error.Kind switch
    {
        ErrorKind.NotFound => Results.NotFound(new { result.Error.Message }),
        ErrorKind.Conflict => Results.Problem(result.Error.Message, statusCode: StatusCodes.Status409Conflict),
        _ => Results.Problem(result.Error.Message, statusCode: StatusCodes.Status400BadRequest),
    };
```

Para um serviço pequeno onde todo erro de negócio vira `400`, a `string` simples basta — adicione a categoria quando o primeiro `404`/`409` aparecer, não preventivamente.

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
