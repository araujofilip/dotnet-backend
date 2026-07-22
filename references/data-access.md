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

Nunca exponha um endpoint de listagem sem paginação ou limite de resultado — `ToListAsync()` sem `Take()` é uma query que funciona bem em dev com 50 registros e derruba o serviço em produção com 5 milhões.

O contrato de resposta é um envelope com os itens e os metadados de página; `pageSize` sempre tem teto validado na borda (default 20, máximo 100 — sem teto, `?pageSize=1000000` é a query sem limite de volta):

```csharp
public sealed record PagedResult<T>(IReadOnlyList<T> Items, int Page, int PageSize, int TotalCount);

// Validator do query: Page > 0, PageSize entre 1 e 100

var page = await db.Orders
    .AsNoTracking()
    .OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Id) // tiebreaker: sem ele, timestamps iguais duplicam/pulam itens entre páginas
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync(ct);
```

A ordenação precisa de índice cobrindo filtro + ordenação (ex.: `(Status, CreatedAt, Id)`); sem ele, a query vira scan em tabela grande. Em Postgres, crie índice em tabela grande com `CREATE INDEX CONCURRENTLY` (migration com SQL manual) para não bloquear escrita.

**Offset vs keyset:** `Skip/Take` degrada em páginas profundas (o banco lê e descarta todas as linhas puladas — página 10.000 × 20 lê 200 mil linhas). Para scroll infinito, exports, ou qualquer consumidor que caminha longe na lista, use keyset pagination — o cursor é o último item visto:

```csharp
// cursor = (CreatedAt, Id) do último item da página anterior
var page = await db.Orders
    .AsNoTracking()
    .Where(o => o.CreatedAt < lastCreatedAt || (o.CreatedAt == lastCreatedAt && o.Id < lastId))
    .OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Id)
    .Take(pageSize)
    .ToListAsync(ct);
```

No contrato HTTP, exponha o cursor como string opaca (ex.: base64 de `"{CreatedAt:o}|{Id}"`) devolvida como `nextCursor` no envelope — o cliente só a devolve na próxima chamada, nunca interpreta; isso permite mudar o esquema do cursor sem quebrar consumidores. Busque `pageSize + 1` itens para saber se existe próxima página sem uma segunda query.

Keyset não permite "pular para a página N" — se a UI é numerada e os usuários ficam nas primeiras páginas, `Skip/Take` com teto é suficiente e mais simples. Escolha pelo padrão de navegação real, não preventivamente.

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

## Concorrência otimista: detectando lost update

O padrão busca-muta-salva dos handlers tem uma janela: dois usuários carregam o mesmo `Order`, cada um muda uma coisa, o segundo `SaveChanges` sobrescreve silenciosamente o primeiro (lost update). Toda entidade que múltiplos usuários podem editar simultaneamente precisa de um **concurrency token** — o EF Core então gera `UPDATE ... WHERE Version = @original`, e a escrita perdedora falha em vez de sobrescrever:

```csharp
// SQL Server: coluna rowversion
builder.Property(o => o.RowVersion).IsRowVersion();

// PostgreSQL: coluna de sistema xmin, sem coluna própria
builder.Property<uint>("xmin").IsRowVersion();   // ou builder.UseXminAsConcurrencyToken()
```

No handler, a falha vira erro de negócio esperado — `409`, nunca `500`:

```csharp
try
{
    await unitOfWork.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException)
{
    return Result<Unit>.Failure(new Error(ErrorKind.Conflict,
        "O pedido foi alterado por outro usuário. Recarregue e tente novamente."));
}
```

Duas armadilhas: (1) o handler que salva precisa carregar a entidade **com tracking** — `AsNoTracking()` quebra a detecção, o EF precisa do valor original do token pra gerar o `WHERE`; (2) não faça retry automático engolindo o conflito (recarregar e reaplicar sem o usuário ver) a menos que a operação seja comprovadamente comutativa — o padrão seguro é devolver `409` e deixar o cliente decidir.
