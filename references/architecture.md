# Arquitetura: Clean Architecture, DDD e CQRS

## Quando usar isso (e quando não usar)

Clean Architecture + DDD + CQRS resolve um problema específico: domínios com regras de negócio ricas, que mudam com frequência, e onde múltiplas partes do sistema (API, jobs, mensageria) precisam reusar a mesma lógica de domínio sem duplicar regras.

Para um CRUD simples (poucas entidades, sem regra de negócio real além de validação de campo), essa estrutura é overhead puro: mais camadas, mais indireção, mais arquivos para a mesma funcionalidade. Nesses casos, uma abordagem mais direta (Minimal API + serviço de aplicação simples + EF Core) é a escolha certa. Ao propor arquitetura, sempre explique essa troca ao invés de aplicar Clean Architecture por padrão.

## Estrutura de camadas

```
src/
├── MyApp.Domain/           # Entidades, Value Objects, Domain Events, interfaces de repositório
├── MyApp.Application/      # Casos de uso (Commands/Queries + Handlers), DTOs, validação
├── MyApp.Infrastructure/   # Implementação de repositórios, EF Core, integrações externas
└── MyApp.Api/               # Controllers/Minimal API endpoints, composição (DI), middlewares
```

Regra de dependência: as setas apontam para dentro. `Domain` não depende de nada. `Application` depende só de `Domain`. `Infrastructure` e `Api` dependem de `Application`/`Domain`, nunca o contrário. Se você perceber uma entidade de domínio referenciando `Microsoft.EntityFrameworkCore`, isso é um sinal de que a camada está vazando.

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

CQRS separa operações que mudam estado (Commands) de operações que só leem (Queries). Isso não exige bancos separados — a maioria dos sistemas usa o mesmo banco para os dois, mas se beneficia de handlers dedicados e simples, cada um fazendo uma coisa:

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
