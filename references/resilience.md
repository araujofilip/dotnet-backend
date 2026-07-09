# Resiliência e tolerância a falha

## Por que isso não é opcional

Todo serviço backend depende de algo fora do seu controle: banco de dados, fila, API de terceiro, outro microsserviço. Esses componentes vão falhar ou ficar lentos em algum momento — a questão não é "se", é "quando", e o que o seu código faz nesse momento. Código sem estratégia de resiliência não falha graciosamente: ele trava threads, esgota connection pools, e transforma a lentidão de uma dependência num incidente no serviço inteiro (efeito cascata).

## Timeout explícito em toda chamada externa

Toda chamada HTTP, de banco, ou de fila precisa de um timeout explícito e razoável para o contexto — nunca confie no timeout padrão do cliente (que às vezes é "infinito"). Um timeout evita que uma dependência lenta prenda recursos (threads, conexões) indefinidamente:

```csharp
services.AddHttpClient<PaymentGatewayClient>(client =>
{
    client.Timeout = TimeSpan.FromSeconds(5);
});
```

## Polly: retry, circuit breaker e timeout compostos

`Microsoft.Extensions.Http.Resilience` (baseado em Polly) é o padrão atual para compor políticas de resiliência em `HttpClient`:

```csharp
services.AddHttpClient<PaymentGatewayClient>()
    .AddResilienceHandler("payment-gateway", builder =>
    {
        builder.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true
        });

        builder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(15)
        });

        builder.AddTimeout(TimeSpan.FromSeconds(5));
    });
```

**Retry** reenvia a chamada em falhas transitórias (timeout, `5xx`, conexão recusada) — nunca em falhas de negócio (`4xx` como "dados inválidos", que vão falhar de novo do mesmo jeito). Use backoff exponencial com jitter para não sincronizar retries de múltiplas instâncias e piorar uma sobrecarga já existente ("thundering herd").

**Circuit breaker** para de tentar chamar um serviço que está claramente fora do ar, depois de um número de falhas consecutivas, e volta a tentar periodicamente (half-open). Isso protege tanto o seu serviço (não fica preso esperando timeouts repetidos) quanto o serviço com problema (não recebe mais tráfego enquanto se recupera).

**Timeout** por tentativa individual, separado do timeout total da operação com retries.

## Idempotência antes de habilitar retry

Retry só é seguro se a operação for idempotente (executar duas vezes tem o mesmo efeito de executar uma vez). Para operações que mudam estado (ex.: "cobrar o cliente", "criar pedido"), garanta idempotência via uma chave de idempotência fornecida pelo cliente ou gerada e persistida antes da tentativa, antes de habilitar retry automático — senão um retry pode duplicar um efeito colateral real (cobrança duplicada, pedido duplicado).

## Fallback e degradação graciosa

Nem toda falha precisa propagar como erro para o usuário final. Quando fizer sentido para o domínio, defina um fallback explícito:

```csharp
builder.AddFallback(new FallbackStrategyOptions<HttpResponseMessage>
{
    FallbackAction = _ => Outcome.FromResultAsValueTask(CachedRecommendationsResponse())
});
```

Exemplo: se o serviço de recomendações personalizadas está fora do ar, mostrar recomendações genéricas em cache é melhor do que quebrar a página inteira.

## Bulkhead: isolar falhas entre dependências

Se um serviço faz chamadas a múltiplas dependências externas, uma dependência lenta não deve conseguir esgotar todos os recursos (threads, conexões) disponíveis para as outras. `AddConcurrencyLimiter` limita quantas chamadas simultâneas cada dependência pode consumir, isolando o impacto de uma degradação a apenas essa dependência.

## Health checks refletindo dependências reais

Um health check que só responde "estou de pé" (liveness) sem checar dependências críticas (readiness) engana o orquestrador (Kubernetes, load balancer), que continua mandando tráfego para uma instância que não consegue de fato processar requisições:

```csharp
services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "postgres")
    .AddCheck<PaymentGatewayHealthCheck>("payment-gateway");
```

Separe liveness (o processo está vivo? reiniciar ajudaria?) de readiness (o serviço está pronto para receber tráfego agora?) — misturar os dois faz o orquestrador reiniciar um serviço saudável só porque uma dependência externa está temporariamente fora, o que não resolve nada.

## O que logar quando algo falha

Ao capturar uma falha de dependência externa, logue com contexto suficiente para diagnosticar sem precisar reproduzir (nome da dependência, operação, tempo decorrido, e se foi timeout/circuit aberto/erro de negócio) — e nunca apenas engula a exceção. Se o circuito abrir, isso deve ser visível em métricas/alertas, não descoberto só quando um usuário reclama.
