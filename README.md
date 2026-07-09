# dotnet-backend-expert

Skill do Claude para atuar como um especialista sênior em backend .NET/C#: código idiomático e legível, Clean Architecture com DDD e CQRS, design de APIs ASP.NET Core, acesso a dados com EF Core, performance e concorrência, resiliência (retry/circuit breaker/timeout), background workers/serviços hospedados, e testes/qualidade de código.

## Estrutura

```
dotnet-backend-expert/
├── SKILL.md                              # ponto de entrada: princípios gerais + roteamento
└── references/
    ├── code-style.md                     # C# idiomático, nullable, records, pattern matching
    ├── architecture.md                   # Clean Architecture, DDD, CQRS, MediatR
    ├── api-design.md                     # ASP.NET Core, Minimal APIs, versionamento, auth
    ├── data-access.md                    # EF Core, N+1, paginação, Dapper
    ├── performance-concurrency.md        # async/await, Channels, Span<T>, BenchmarkDotNet
    ├── resilience.md                     # Polly: retry, circuit breaker, timeout, bulkhead
    ├── background-workers.md             # BackgroundService, filas, Hangfire/Quartz.NET
    └── testing-quality.md                # xUnit, TestContainers, CRAP score, quality gates
```

## Instalação

### Claude Code (CLI)

Clone este repositório e copie a pasta da skill para o diretório de skills do seu projeto ou global.

**Por projeto** (recomendado — vale só para este repositório):

```bash
git clone https://github.com/araujofilip/dotnet-backend-expert.git /tmp/dotnet-backend-expert
mkdir -p .claude/skills
cp -r /tmp/dotnet-backend-expert/dotnet-backend-expert .claude/skills/
```

**Global** (disponível em todos os seus projetos .NET):

```bash
git clone https://github.com/araujofilip/dotnet-backend-expert.git /tmp/dotnet-backend-expert
mkdir -p ~/.claude/skills
cp -r /tmp/dotnet-backend-expert/dotnet-backend-expert ~/.claude/skills/
```

O Claude Code carrega qualquer skill presente em `.claude/skills/<nome>/SKILL.md` automaticamente — não precisa de comando de instalação adicional.

### Claude.ai / Cowork

1. Baixe o arquivo compactado da skill (`dotnet-backend-expert.skill` ou `.zip`) deste repositório, ou gere um novo com:
   ```bash
   cd dotnet-backend-expert
   zip -r ../dotnet-backend-expert.skill . -x ".git/*"
   ```
2. Envie o arquivo `.skill` numa conversa do Claude (Cowork ou claude.ai) e clique em **Salvar skill** no card que aparece.

### GitHub Copilot

```bash
git clone https://github.com/araujofilip/dotnet-backend-expert.git /tmp/dotnet-backend-expert
mkdir -p .github/skills
cp -r /tmp/dotnet-backend-expert/dotnet-backend-expert/* .github/skills/
```

### OpenCode

```bash
git clone https://github.com/araujofilip/dotnet-backend-expert.git /tmp/dotnet-backend-expert
mkdir -p ~/.config/opencode/skills/dotnet-backend-expert
cp /tmp/dotnet-backend-expert/dotnet-backend-expert/SKILL.md ~/.config/opencode/skills/dotnet-backend-expert/
cp -r /tmp/dotnet-backend-expert/dotnet-backend-expert/references ~/.config/opencode/skills/dotnet-backend-expert/
```

## Uso

A skill dispara automaticamente quando você pede para escrever, revisar, arquitetar ou refatorar código backend em .NET/C# — APIs, workers, camadas de domínio, repositórios, migrations, handlers CQRS, etc. Também pode ser invocada explicitamente com `/dotnet-backend-expert` (Claude Code) ou mencionando a skill pelo nome.

## Licença

MIT — adapte livremente para o seu contexto.
