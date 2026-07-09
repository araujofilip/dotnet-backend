# Performance e concorrência

## Meça antes de otimizar

Nada nesta referência deve ser aplicado "por precaução" em código que não está num hot path. Otimização prematura tem custo real: código mais difícil de ler, mais difícil de manter. Use o profiler (Visual Studio Profiler, `dotnet-trace`) ou BenchmarkDotNet para confirmar que algo é de fato um gargalo antes de complicar o código por causa dele.

## Async/await: o básico que sempre importa

- Nunca use `async void`, exceto em event handlers de UI — exceções não capturadas em `async void` derrubam o processo em vez de propagar para quem chamou.
- Propague `CancellationToken` em toda a cadeia de chamadas assíncronas (repositório, handler, chamada HTTP) — sem isso, cancelar uma requisição no cliente não cancela o trabalho no servidor, e recursos continuam sendo gastos à toa.
- Evite `.Result` ou `.Wait()` em código assíncrono — isso pode causar deadlock em contextos com `SynchronizationContext` (menos comum em ASP.NET Core moderno, mas ainda um risco real em bibliotecas reusadas).
- Use `ConfigureAwait(false)` em bibliotecas de propósito geral (não em código de aplicação ASP.NET Core, onde não há `SynchronizationContext` a evitar).

## Task vs Channel vs lock vs actor

| Cenário | Ferramenta |
|---|---|
| Uma operação assíncrona isolada (chamada HTTP, query) | `Task`/`async-await` puro |
| Produtor(es)/consumidor(es) com backpressure (processar itens de uma fila em background) | `System.Threading.Channels` |
| Proteger uma seção crítica curta e síncrona (contador em memória, cache local) | `lock` (ou `SemaphoreSlim` se a seção crítica for assíncrona) |
| Estado compartilhado complexo, com muitas transições concorrentes (ex.: coordenação de múltiplos workers) | Considere um modelo de ator (Akka.NET) em vez de reinventar sincronização manual — é mais fácil de raciocinar sobre correção do que uma teia de locks |

`lock` nunca deve envolver uma chamada assíncrona dentro do bloco (`lock { await ... }` nem compila, e mesmo contornando isso indiretamente é um sinal de design errado) — use `SemaphoreSlim.WaitAsync()` quando a seção crítica precisar ser assíncrona.

## Processamento paralelo

Para processar uma coleção com uma operação assíncrona por item, prefira `Parallel.ForEachAsync` a `Task.WhenAll(items.Select(...))` sem limite — o segundo pode disparar centenas de chamadas simultâneas a um serviço externo e derrubá-lo:

```csharp
await Parallel.ForEachAsync(orderIds, new ParallelOptions { MaxDegreeOfParallelism = 8 }, async (id, ct) =>
{
    await ProcessOrderAsync(id, ct);
});
```

## Alocação de memória em hot paths

Em código executado com muita frequência (middleware, parsing, serialização customizada), considere:

- `Span<T>`/`ReadOnlySpan<T>` para evitar alocações intermediárias ao manipular strings/arrays.
- `ArrayPool<T>.Shared` para buffers reusáveis em vez de `new byte[size]` a cada chamada.
- `ValueTask<T>` em vez de `Task<T>` quando o resultado frequentemente já está disponível de forma síncrona (ex.: cache hit) — evita alocar um `Task` novo em cada chamada.

Novamente: isso é para hot paths medidos, não para todo método assíncrono do sistema.

## Cache

Cache é uma ferramenta de performance com um custo escondido: dados potencialmente desatualizados. Antes de adicionar cache, decida explicitamente a política de invalidação (TTL, invalidação ativa no `Update`, ou ambos) — cache sem estratégia de invalidação clara é uma fonte comum de bugs "só acontece às vezes" em produção.

- `IMemoryCache` para cache local de processo único.
- `IDistributedCache` (Redis) quando múltiplas instâncias precisam compartilhar o cache ou sobreviver a restart.

## AOT e trimming

Para serviços onde tempo de cold start importa (ex.: Azure Functions, containers que escalam para zero), avalie Native AOT — mas isso restringe o uso de reflection não anotada e alguns pacotes de terceiros. É uma decisão de arquitetura a se tomar cedo, não uma otimização de última hora.

## BenchmarkDotNet para decisões de performance

Quando a dúvida for "abordagem A ou B é mais rápida", não confie em intuição nem em medição manual com `Stopwatch` — use BenchmarkDotNet, que já lida com warmup, GC e variância estatística:

```csharp
[MemoryDiagnoser]
public class SerializationBenchmarks
{
    [Benchmark(Baseline = true)]
    public byte[] SystemTextJson() => JsonSerializer.SerializeToUtf8Bytes(_order);

    [Benchmark]
    public byte[] MessagePack() => MessagePackSerializer.Serialize(_order);
}
```
