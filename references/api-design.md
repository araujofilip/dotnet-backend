# Design de APIs com ASP.NET Core

## Minimal APIs vs Controllers

Minimal APIs são a escolha padrão para serviços novos e focados (poucos endpoints, sem muita infraestrutura compartilhada de filtro/binding customizado) — menos boilerplate, startup mais rápido, e mais fácil de testar isoladamente. Controllers ainda fazem sentido quando o projeto já usa bastante infraestrutura de MVC (model binding customizado complexo, convenções de rota elaboradas, ou uma equipe já familiarizada com esse estilo).

```csharp
app.MapPost("/orders", async (CreateOrderRequest request, ISender mediator, CancellationToken ct) =>
{
    var result = await mediator.Send(new CreateOrderCommand(request.CustomerId, request.Lines), ct);

    return result.IsSuccess
        ? Results.Created($"/orders/{result.Value}", result.Value)
        : Results.Problem(result.Error, statusCode: StatusCodes.Status400BadRequest);
})
.WithName("CreateOrder")
.Produces<Guid>(StatusCodes.Status201Created)
.ProducesValidationProblem();
```

Agrupe endpoints relacionados com `RouteGroupBuilder` para compartilhar prefixo, tags de OpenAPI e filtros:

```csharp
var orders = app.MapGroup("/orders").WithTags("Orders").RequireAuthorization();
orders.MapPost("/", CreateOrder);
orders.MapGet("/{id:guid}", GetOrder);
```

## O endpoint não deve conter lógica de negócio

O corpo do endpoint deve só: (1) traduzir a requisição HTTP para um Command/Query, (2) despachar para o handler (via MediatR ou serviço de aplicação), (3) traduzir o resultado para uma resposta HTTP. Qualquer decisão de negócio (validação de regra, cálculo) pertence à camada de aplicação/domínio — isso é o que permite testar a lógica sem subir um `WebApplicationFactory`.

## Validação

Valide a forma dos dados de entrada (campos obrigatórios, formato) na borda, antes de chegar à lógica de negócio — com FluentValidation ou Data Annotations + `IEndpointFilter`. Retorne `400 Bad Request` com detalhes estruturados (RFC 7807 `ProblemDetails`), não uma mensagem de texto solta:

```csharp
app.MapPost("/orders", CreateOrder)
   .AddEndpointFilter<ValidationFilter<CreateOrderRequest>>();
```

## Versionamento

Toda API que pode ter consumidores externos (mobile, terceiros, outro time) precisa de uma estratégia de versionamento desde o primeiro endpoint — adicionar depois é muito mais caro. Opções comuns: versionamento por URL (`/v1/orders`) é o mais simples de entender e cachear; por header é mais "limpo" mas exige mais disciplina de client. Use o pacote `Asp.Versioning.Http` para Minimal APIs.

Regra prática vinda do princípio de "extend-only": depois que um endpoint é publicado, só adicione campos opcionais — nunca remova ou mude o significado de um campo existente numa versão já publicada. Se precisar de uma mudança incompatível, crie uma nova versão.

## Autenticação e autorização

- Autenticação (quem é o usuário) via JWT Bearer é o padrão para APIs stateless. Configure `AddAuthentication().AddJwtBearer(...)` e valide issuer, audience e tempo de expiração — não desabilite validação "temporariamente" em produção.
- Autorização (o que o usuário pode fazer) via policies (`RequireAuthorization("CanManageOrders")`), não via `if (user.Role == "Admin")` espalhado pelo código. Policies centralizam a regra e são testáveis isoladamente.
- Nunca devolva detalhes de erro de autenticação que ajudem um atacante a enumerar usuários válidos (ex.: "usuário não existe" vs "senha errada" — use uma mensagem genérica para os dois).

## Contratos de resposta consistentes

- Sucesso: `200`/`201`/`204` conforme a operação (criação retorna `201` com `Location`; deleção sem corpo retorna `204`).
- Erro de cliente (validação, não encontrado, conflito): `4xx` com `ProblemDetails`.
- Erro inesperado: `500`, sem vazar stack trace ou detalhes internos — logue o detalhe internamente, devolva algo genérico e um `traceId` para correlação com os logs.

## Rate limiting e proteção básica

Para endpoints públicos ou de alto custo (ex.: geração de relatório, login), configure `Microsoft.AspNetCore.RateLimiting` desde o início — não é algo para "adicionar depois que tiver problema", porque nesse ponto já é um incidente:

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("login", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 5;
    });
});
```

## Observabilidade do endpoint

Todo endpoint deve ser observável sem precisar reproduzir o problema localmente: health checks (`/health`) para liveness/readiness, OpenTelemetry para tracing distribuído (essencial se há chamadas a outros serviços), e logging estruturado com `ILogger<T>` incluindo IDs de correlação — nunca `Console.WriteLine` ou interpolação de string em mensagens de log (use os parâmetros do `ILogger` para permitir logs estruturados e pesquisáveis).

```csharp
logger.LogInformation("Pedido {OrderId} criado para cliente {CustomerId}", orderId, customerId);
```
