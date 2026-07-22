---
name: dotnet-backend
description: Use quando o usuário pedir para escrever, revisar, arquitetar ou refatorar código backend em .NET/C# — API ASP.NET Core, worker, consumidor de fila, job agendado, handler CQRS, camada de domínio, repositório, migration EF Core — ou perguntar sobre boas práticas, arquitetura, resiliência, performance ou testes em contexto C#/.NET, mesmo sem mencionar ".NET" explicitamente.
---

# Backend .NET

Referência para produzir e revisar código backend .NET que uma equipe experiente aprovaria em code review: legível, testável, com arquitetura proporcional ao tamanho do problema, resiliente a falhas de dependências externas e sem armadilhas de performance conhecidas.

Este arquivo resume os princípios que valem para qualquer tarefa; o detalhamento de cada área está em `references/`. Leia a referência da área antes de escrever código nela — cada arquivo tem exemplos concretos, padrões recomendados e anti-padrões. Ao revisar código existente, use as referências como checklist e aponte desvios explicando o porquê (não só "isso está errado").

| Se a tarefa envolve... | Leia |
|---|---|
| Legibilidade, nomenclatura, C# idiomático, records, pattern matching, nullable reference types | `references/code-style.md` |
| Organização de projeto/solução, camadas, Vertical Slice, DDD, CQRS, MediatR, Domain Events, Result pattern | `references/architecture.md` |
| Endpoints, Minimal APIs vs Controllers, validação, versionamento, autenticação/autorização | `references/api-design.md` |
| EF Core, migrations, queries, N+1, leitura em alta escala, Dapper | `references/data-access.md` |
| Async/await, concorrência, alocação de memória, Span<T>, benchmarks | `references/performance-concurrency.md` |
| Chamadas a serviços externos, timeouts, retries, circuit breaker, resiliência | `references/resilience.md` |
| Background worker, `BackgroundService`/`IHostedService`, consumidor de fila, job agendado (Hangfire/Quartz) | `references/background-workers.md` |
| **Qualquer geração de código de produção novo (leitura obrigatória, sempre)** — além de cobertura, qualidade de código, detecção de anti-padrões | `references/testing-quality.md` |

Não é preciso ler tudo de uma vez — trate cada referência como material sob demanda para a parte da tarefa que a exige. Exceção: `references/testing-quality.md` faz parte de toda tarefa que gera código de produção novo, porque a entrega inclui os testes.

## Princípios sempre válidos

Independente da referência específica, estes princípios guiam qualquer código backend .NET produzido ou revisado com esta skill:

**Legibilidade antes de esperteza.** Código backend é lido dezenas de vezes para cada vez que é escrito. Prefira nomes explícitos, métodos pequenos com um propósito claro, e fluxo de controle direto (early returns em vez de aninhamento profundo de `if`). Uma solução "inteligente" que exige releitura cuidadosa para entender geralmente é a solução errada, mesmo que seja mais curta.

**Imutabilidade e tipos que não mentem.** Prefira `record`/`record struct` para modelos de dados, `readonly` para campos que não mudam, e nullable reference types habilitado e respeitado (não silenciado com `!` por conveniência). Um tipo que não permite representar um estado inválido evita uma classe inteira de bugs em produção — isso vale mais do que economizar algumas linhas.

**Explicitação de dependências.** Toda dependência externa (banco, fila, serviço HTTP, relógio do sistema) deve entrar via injeção de dependência, nunca via singleton estático ou `new` direto dentro de lógica de negócio. É o que torna o código testável sem mocks mirabolantes.

**Assuma que dependências externas vão falhar.** Toda chamada de rede (HTTP, banco, fila) deve ter timeout explícito e uma estratégia consciente para falha (retry com backoff, circuit breaker, fallback, ou falha rápida e visível). "Deixar sem tratamento e torcer" não é uma estratégia — veja `references/resilience.md`.

**Escolha a arquitetura para o tamanho do problema, não para o currículo.** Clean Architecture + DDD + CQRS com MediatR é a resposta certa para domínios complexos com regras de negócio ricas — não para um CRUD simples. Ao propor arquitetura, explique a troca (mais camadas = mais indireção, mas também mais testabilidade e separação de responsabilidades) em vez de aplicar o padrão por reflexo. Veja `references/architecture.md`.

**Entrega de código novo inclui os testes.** Ao gerar código de produção novo, a entrega contém: (1) o código, (2) o(s) arquivo(s) de teste cobrindo a regra de negócio ou o comportamento observável do endpoint/worker/handler, e (3) o projeto de teste (`.csproj`) se ainda não existir. Não é sobre 100% de cobertura — é um teste que quebra se alguém quebrar o comportamento. Leia `references/testing-quality.md` para o que testar e como.

**Sem "mágica" desnecessária.** Evite reflection pesada, AutoMapper e frameworks que escondem o que o código realmente faz. Use mapeamento explícito (construtor, método `ToDto()`) — é mais fácil de debugar e o compilador ajuda. Se o projeto existente já usa um desses frameworks de forma consolidada, siga o padrão do projeto em vez de misturar estilos.

## Ao revisar código existente

Ao invés de listar tudo que está "errado", priorize por impacto: primeiro correção (o código faz o que deveria, inclusive nos caminhos de erro?), depois segurança e resiliência a falhas externas, depois legibilidade/manutenibilidade, depois performance. Só levante performance como prioridade alta se houver evidência real de que é um gargalo (hot path, dado de profiling, alto volume) — otimização prematura tem custo real em legibilidade.
