# Background Workers e serviços hospedados

## Quando é um worker, não uma API

Um background worker é o processo certo quando o trabalho não é iniciado por uma requisição HTTP síncrona: consumir mensagens de uma fila, rodar um job agendado (fechamento de fatura à meia-noite), ou processar um fluxo contínuo de eventos. Se o "cliente" está esperando uma resposta imediata, é um endpoint de API, não um worker — não force um padrão fire-and-forget onde o usuário precisa do resultado na hora.

## `BackgroundService`: o ponto de partida

`BackgroundService` (que implementa `IHostedService`) é a base para praticamente todo worker em .NET. O host chama `StartAsync` no boot e `StopAsync` no shutdown — sua lógica principal vive em `ExecuteAsync`:

```csharp
public sealed class OrderProcessingWorker(
    IServiceScopeFactory scopeFactory,
    ILogger<OrderProcessingWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var processor = scope.ServiceProvider.GetRequiredService<IOrderProcessor>();
                await processor.ProcessPendingOrdersAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break; // shutdown normal, não é erro
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Falha ao processar lote de pedidos pendentes");
                // decida conscientemente: continuar no próximo ciclo, ou propagar e deixar o processo reiniciar
            }

            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }
}
```

Pontos que sempre importam aqui:

- **`ExecuteAsync` não deve deixar uma exceção não tratada vazar** de forma que mate o loop inteiro silenciosamente — decida explicitamente se um erro num ciclo deve ser logado e seguir para o próximo, ou se é grave o suficiente para deixar o `IHostedService` falhar (nesse caso, configure `Host.CreateApplicationBuilder` para que uma falha de hosted service derrube o processo — `BackgroundServiceExceptionBehavior.StopHost` — para que o orquestrador reinicie o worker em vez de deixá-lo "vivo" mas parado de processar).
- **`IServiceScopeFactory`, não injetar serviços scoped direto no worker.** `BackgroundService` é singleton por natureza (roda uma vez, vive o processo inteiro); um `DbContext` (scoped) injetado direto no construtor do worker seria compartilhado entre todas as iterações do loop, o que causa bugs de estado sutis. Crie um `scope` novo por unidade de trabalho.
- **`stoppingToken` propagado em tudo.** Toda chamada assíncrona dentro do loop (query, chamada HTTP, delay) recebe o token, para que o shutdown seja rápido e não force um `kill -9` do orquestrador.

## Graceful shutdown

Quando o host recebe um sinal de shutdown (SIGTERM em container, `Ctrl+C` local), ele chama `StopAsync` em todos os `IHostedService`, com um timeout configurável (`HostOptions.ShutdownTimeout`, padrão 5s — aumente se o worker processa itens que não podem ser interrompidos no meio, como escrever um arquivo grande). O objetivo é terminar a unidade de trabalho em andamento (ou desistir dela de forma segura) antes do processo ser derrubado à força.

```csharp
builder.Services.Configure<HostOptions>(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
});
```

Se o worker está no meio de processar um item quando o `stoppingToken` é cancelado, a decisão de negócio importa: para consumo de fila, normalmente é melhor não confirmar (ack) a mensagem e deixá-la voltar para a fila, para outro consumidor pegar depois — não tente "forçar" terminar o processamento durante o shutdown.

## Consumindo filas (RabbitMQ, Azure Service Bus, SQS)

O padrão geral: receber a mensagem, processar, só confirmar (ack) depois que o processamento (incluindo qualquer persistência) foi concluído com sucesso. Confirmar antes (auto-ack) significa perder a mensagem se o processo cair no meio do trabalho.

```csharp
public sealed class OrderQueueConsumer(
    IServiceScopeFactory scopeFactory,
    ILogger<OrderQueueConsumer> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var message in _receiver.ReceiveMessagesAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var handler = scope.ServiceProvider.GetRequiredService<IOrderMessageHandler>();
                await handler.HandleAsync(message.Body, stoppingToken);

                await _receiver.CompleteMessageAsync(message, stoppingToken); // ack
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Falha ao processar mensagem {MessageId}", message.MessageId);
                await _receiver.AbandonMessageAsync(message, cancellationToken: stoppingToken); // nack, volta pra fila
            }
        }
    }
}
```

Pontos que sempre importam:

- **Idempotência é obrigatória, não opcional.** Toda fila com pelo menos uma entrega garantida (a norma) pode entregar a mesma mensagem mais de uma vez (reconexão, timeout de ack, rebalanceamento). O handler precisa ser seguro para processar a mesma mensagem duas vezes — normalmente via uma chave de idempotência (ID da mensagem) verificada antes de aplicar o efeito.
- **Dead-letter queue para mensagens que falham repetidamente.** Depois de N tentativas malsucedidas, mova a mensagem para uma fila de erro em vez de deixá-la reentregando para sempre (isso trava o processamento de mensagens saudáveis atrás dela, e mascara o problema). A maioria dos brokers (Service Bus, RabbitMQ com política de DLQ) faz isso nativamente — configure em vez de reimplementar.
- **Não processe mensagens em paralelo sem limite.** Configure `MaxConcurrentCalls` (Service Bus) ou `PrefetchCount`/QoS (RabbitMQ) de forma consciente — processamento paralelo ilimitado pode sobrecarregar o mesmo banco/serviço externo que o resto do sistema usa (ver `resilience.md` sobre bulkhead).

## Jobs agendados: Hangfire vs Quartz.NET vs `PeriodicTimer`

| Necessidade | Ferramenta |
|---|---|
| Loop simples de polling em intervalo fixo, sem persistência de agendamento entre restarts | `PeriodicTimer` dentro de um `BackgroundService` (como no primeiro exemplo) |
| Jobs agendados (cron), com dashboard, retry automático, persistência do agendamento em banco | **Hangfire** — mais simples de configurar, bom para a maioria dos casos de "rodar isso todo dia às 2h" |
| Agendamento mais sofisticado (múltiplos triggers, jobs com dependência entre si, clustering explícito) | **Quartz.NET** — mais poder, mais configuração |

```csharp
// Hangfire
RecurringJob.AddOrUpdate<IInvoiceCloser>(
    "close-daily-invoices",
    closer => closer.CloseAsync(CancellationToken.None),
    Cron.Daily(2));
```

**Todo job agendado precisa ser idempotente e seguro contra execução sobreposta.** Se o job de fechamento de fatura demorar mais que o intervalo entre execuções, ou se houver mais de uma instância do worker rodando (deploy com múltiplas réplicas), duas execuções podem rodar ao mesmo tempo. Hangfire e Quartz.NET têm mecanismos de lock distribuído para isso — habilite-os (não assuma "só vai rodar uma instância" como garantia, porque orquestradores fazem rolling deploy com duas réplicas temporariamente ativas).

## Produtor/consumidor em processo com `Channels`

Para desacoplar quem gera trabalho de quem o processa dentro do mesmo processo (sem fila externa), `System.Threading.Channels` dá backpressure de graça — se o consumidor não acompanha, o produtor é bloqueado em vez de acumular memória sem limite:

```csharp
var channel = Channel.CreateBounded<OrderId>(new BoundedChannelOptions(capacity: 100)
{
    FullMode = BoundedChannelFullMode.Wait
});
```

Ver `references/performance-concurrency.md` para o detalhamento de `Channels` vs outras ferramentas de concorrência.

## Observabilidade específica de worker

Um worker não tem requisições HTTP para monitorar latência — a observabilidade certa é diferente da de uma API:

- **Heartbeat/liveness real**: exponha uma métrica ou health check que reflita "o loop principal ainda está rodando e processou algo recentemente", não apenas "o processo está de pé". Um worker travado num deadlock ainda responde "estou vivo" num health check ingênuo.
- **Métricas de fila/lag**: para consumidores de fila, monitore o tamanho da fila e a idade da mensagem mais antiga não processada — isso indica se o worker está acompanhando o volume, muito antes de virar um incidente.
- **Log estruturado por unidade de trabalho**, com um ID de correlação (ID da mensagem, ID do job) propagado, igual à recomendação de `api-design.md` para requisições HTTP.

## Testando workers

Teste a lógica de processamento (`IOrderProcessor`, `IOrderMessageHandler`) isoladamente, como qualquer outro serviço de aplicação — sem precisar instanciar o `BackgroundService` inteiro. Para testar o comportamento de idempotência e de retry/dead-letter, um teste de integração com um broker real em container (`Testcontainers.RabbitMq` — ver `references/testing-quality.md`) é o que realmente valida que ack/nack e redelivery funcionam como esperado; mocks de fila não pegam esse tipo de erro de configuração.
