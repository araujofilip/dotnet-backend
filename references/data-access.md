# Acesso a dados: EF Core e leitura em escala

## Configuração de entidades

Use `IEntityTypeConfiguration<T>` por entidade em vez de Data Annotations ou um `OnModelCreating` gigante — mantém a configuração de cada entidade num lugar previsível e testável:

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
            .HasConversion(id => id.Value, value => new OrderId(value));

        builder.OwnsMany(o => o.Lines, lines =>
        {
            lines.WithOwner().HasForeignKey("OrderId");
            lines.Property(l => l.Quantity).IsRequired();
        });

        builder.Navigation(o => o.Lines).UsePropertyAccessMode(PropertyAccessMode.Field);
    }
}
```

## Migrations

- Uma migration por mudança lógica, com nome descritivo (`AddOrderStatusIndex`, não `Migration1`).
- Revise o SQL gerado (`dotnet ef migrations script`) antes de aplicar em produção, especialmente para mudanças em tabelas grandes — um `ALTER TABLE` que parece inofensivo pode travar a tabela inteira dependendo do volume de dados e do provedor.
- Nunca edite uma migration já aplicada em produção; crie uma nova migration corretiva.

## Evitando N+1

O erro mais comum em EF Core é iterar uma coleção e disparar uma query por item. Sempre que uma query vai acessar dados relacionados, use `Include`/`ThenInclude` explicitamente ou, melhor ainda, projete diretamente para o DTO que a query realmente precisa:

```csharp
// N+1: uma query por pedido para buscar as linhas
var orders = await db.Orders.ToListAsync(ct);
foreach (var order in orders)
{
    var lines = order.Lines; // dispara lazy loading, uma query por pedido
}

// Correto: projeção única, sem lazy loading
var orderSummaries = await db.Orders
    .Select(o => new OrderSummaryDto(o.Id, o.CustomerId, o.Lines.Count, o.Lines.Sum(l => l.Quantity * l.UnitPrice)))
    .ToListAsync(ct);
```

Desabilite lazy loading por padrão no projeto (não instale `Microsoft.EntityFrameworkCore.Proxies`) — ele torna N+1 invisível no código-fonte, só aparece no log de SQL ou em produção sob carga.

## AsNoTracking para leitura

Toda query que só lê dados (não vai fazer `SaveChanges` depois) deve usar `AsNoTracking()` — evita o overhead do change tracker do EF Core, que só é necessário quando você vai modificar a entidade:

```csharp
var orders = await db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(ct);
```

## Paginação e limites

Nunca exponha um endpoint de listagem sem paginação ou limite de resultado — `ToListAsync()` sem `Take()` é uma query que funciona bem em dev com 50 registros e derruba o serviço em produção com 5 milhões:

```csharp
var page = await db.Orders
    .AsNoTracking()
    .OrderByDescending(o => o.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync(ct);
```

## Separação leitura/escrita (writes com EF Core, reads com Dapper)

Para a maioria dos casos, EF Core com projeções (`Select` + `AsNoTracking`) já é suficiente para leitura. Considere Dapper especificamente para queries de leitura complexas, de alto volume, ou que exigem SQL muito específico (relatórios, dashboards, agregações pesadas) onde o overhead de tradução do LINQ para SQL do EF Core, ou a dificuldade de expressar a query ideal via LINQ, se torna um problema real e medido — não como padrão automático para toda leitura.

## Queries compiladas para hot paths

Para queries executadas com muita frequência (ex.: buscar um pedido por ID em todo request), `EF.CompileAsyncQuery` evita reconstruir a árvore de expressão a cada chamada:

```csharp
private static readonly Func<AppDbContext, OrderId, CancellationToken, Task<Order?>> GetByIdCompiled =
    EF.CompileAsyncQuery((AppDbContext db, OrderId id, CancellationToken ct) =>
        db.Orders.FirstOrDefault(o => o.Id == id));
```

Só vale a complexidade extra se o perfil de uso realmente justificar — meça antes de otimizar (ver `performance-concurrency.md`).

## Transações e Unit of Work

Uma operação que muda múltiplas entidades relacionadas deve ser uma única transação — o `DbContext` já atua como Unit of Work por padrão dentro de um único `SaveChangesAsync()`. Para operações que precisam de múltiplas chamadas a `SaveChanges` dentro da mesma unidade lógica, use `db.Database.BeginTransactionAsync()` explicitamente, e sempre com `CancellationToken` propagado.
