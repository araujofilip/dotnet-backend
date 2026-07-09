# Testes e qualidade de código

## O que testar (e o que não vale a pena testar)

Priorize testes que quebram quando um comportamento real quebra: regras de negócio no domínio (`Order.Submit()` lança exceção se não houver itens), handlers de Command/Query, e o comportamento observável de um endpoint (status code, forma da resposta). Não persiga cobertura de propriedades triviais (getters, DTOs sem lógica) — esses testes custam manutenção sem detectar bugs reais.

## Pirâmide de testes para um serviço .NET típico

- **Testes unitários (xUnit)**: entidades de domínio e handlers, com dependências externas mockadas (`NSubstitute`/`Moq`) ou, melhor ainda, com implementações in-memory simples quando a interface é pequena. Rápidos, rodam em milissegundos, formam a maior parte da suíte.
- **Testes de integração (TestContainers)**: repositórios reais contra um banco real (Postgres, via container Docker efêmero), garantindo que a configuração do EF Core e as queries funcionam de verdade — mocks de repositório não pegam erro de mapeamento ou de query mal formada.
- **Testes de API (WebApplicationFactory)**: sobem o pipeline HTTP inteiro em memória e testam o comportamento fim a fim de um endpoint, incluindo middlewares, validação e serialização.

```csharp
public class OrderTests
{
    [Fact]
    public void Submit_WithNoLines_ThrowsInvalidOperation()
    {
        var order = Order.Create(CustomerId.New());

        var act = () => order.Submit();

        Assert.Throws<InvalidOperationException>(act);
    }
}
```

```csharp
public class OrderRepositoryTests : IClassFixture<PostgresContainerFixture>
{
    [Fact]
    public async Task GetByIdAsync_ReturnsOrderWithLines()
    {
        await using var db = _fixture.CreateDbContext();
        var repository = new OrderRepository(db);
        var order = Order.Create(CustomerId.New());
        order.AddLine(ProductId.New(), 2, new Money(10, "BRL"));
        repository.Add(order);
        await db.SaveChangesAsync();

        var loaded = await repository.GetByIdAsync(order.Id, CancellationToken.None);

        Assert.NotNull(loaded);
        Assert.Single(loaded.Lines);
    }
}
```

## TestContainers em vez de banco/fila compartilhado

Nunca dependa de um banco de dados compartilhado e persistente para testes de integração — testes ficam frágeis (um teste deixa dado sujo para o próximo) e não rodam de forma confiável em paralelo ou em CI. `Testcontainers.PostgreSql`/`Testcontainers.RabbitMq` sobem um container efêmero por execução, isolado e descartável.

## Testes de contrato/snapshot para respostas de API

Para respostas de API com forma complexa (ex.: relatórios, payloads de webhook), snapshot testing (biblioteca `Verify`) evita reescrever asserções gigantes manualmente e torna mudanças de forma explícitas e revisáveis no code review:

```csharp
[Fact]
public Task GetOrder_ReturnsExpectedShape()
{
    var response = await client.GetAsync("/orders/123");
    return Verify(await response.Content.ReadAsStringAsync());
}
```

Uma mudança na resposta gera um diff explícito no snapshot — se for intencional, aprova-se o novo snapshot; se não for, o teste pegou uma regressão antes de chegar em produção.

## Cobertura como sinal, não como meta

Cobertura de linha (`%`) sozinha é um sinal fraco — 90% de cobertura não significa que os caminhos de erro importantes foram testados. Prefira olhar cobertura por área crítica do domínio, e complementar com **CRAP score** (Change Risk Anti-Patterns), que combina complexidade ciclomática com cobertura para identificar especificamente o código complexo E não testado — que é onde bugs realmente se escondem.

## Quality gates automatizados

- Analisadores estáticos (`.editorconfig` com regras de análise, `dotnet format --verify-no-changes` no CI) pegam problemas de estilo e alguns bugs antes do code review humano.
- Para código gerado ou fortemente assistido por IA, vale rodar uma checagem específica de anti-padrões comuns desse tipo de geração (código morto deixado para trás, tratamento de exceção genérico demais, testes que sempre passam porque não afirmam nada de fato) — trate isso como parte do quality gate, não como revisão manual opcional.
- CI deve falhar o build em: testes quebrados, cobertura abaixo do limiar acordado pela equipe nas áreas críticas, e avisos de nullable reference type não tratados.

## Test data builders em vez de objetos gigantes inline

Para entidades com muitos campos obrigatórios, um builder com valores padrão sensatos deixa cada teste focado só no que ele realmente varia:

```csharp
public sealed class OrderBuilder
{
    private CustomerId _customerId = CustomerId.New();
    private readonly List<(ProductId, int, Money)> _lines = [];

    public OrderBuilder WithCustomer(CustomerId customerId) { _customerId = customerId; return this; }
    public OrderBuilder WithLine(ProductId productId, int quantity, Money price)
    {
        _lines.Add((productId, quantity, price));
        return this;
    }

    public Order Build()
    {
        var order = Order.Create(_customerId);
        foreach (var (productId, quantity, price) in _lines)
            order.AddLine(productId, quantity, price);
        return order;
    }
}
```
