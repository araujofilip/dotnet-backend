# Código legível e C# idiomático

Esta referência cobre como escrever C# moderno que uma equipe sênior aprovaria em code review sem comentários de estilo.

## Nomenclatura e forma

- Nomes de métodos são verbos (`CalculateTotal`, não `Calculation`); nomes de tipos são substantivos.
- Evite abreviações não óbvias (`custId` → `customerId`). O ganho de digitação não compensa o custo de leitura.
- Prefira métodos curtos com um único nível de abstração. Se um método mistura "orquestrar o fluxo" com "calcular um valor específico", extraia o cálculo.
- Use early return para reduzir aninhamento:

```csharp
// Evite
public decimal CalculateDiscount(Order order)
{
    if (order != null)
    {
        if (order.Total > 100)
        {
            return order.Total * 0.1m;
        }
    }
    return 0;
}

// Prefira
public decimal CalculateDiscount(Order? order)
{
    if (order is null) return 0;
    if (order.Total <= 100) return 0;

    return order.Total * 0.1m;
}
```

## Nullable reference types

Sempre habilite `<Nullable>enable</Nullable>` no `.csproj` e trate os avisos como reais, não como ruído a ser silenciado com `!`. O ponto de nullable reference types é que o compilador te avisa exatamente onde um `NullReferenceException` pode acontecer em produção — silenciar isso com `!` sem justificativa é reintroduzir o bug que a feature existe para prevenir.

Quando `!` for genuinamente necessário (ex.: você sabe algo que o compilador não sabe), justifique com um comentário curto.

## Records e imutabilidade

Prefira `record`/`record struct` para DTOs, mensagens, eventos de domínio e qualquer tipo que representa dados, não comportamento. Ganhos: igualdade estrutural de graça, `with` para cópias modificadas, e sinalização clara de que o tipo não deve ser mutado depois de criado.

```csharp
public sealed record OrderCreated(Guid OrderId, Guid CustomerId, decimal Total, DateTimeOffset OccurredAt);

public sealed record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0, currency);
}
```

Para entidades de domínio com identidade e ciclo de vida (não apenas dados), uma classe normal ainda é apropriada — mas prefira propriedades com `private set` ou métodos explícitos em vez de setters públicos livres, para que mudanças de estado só aconteçam através de comportamento com significado de negócio (ver `architecture.md`).

## Pattern matching e fluxo de controle

Pattern matching deixa decisões baseadas em tipo/forma explícitas e exaustivas:

```csharp
public string DescribeStatus(OrderStatus status) => status switch
{
    OrderStatus.Pending => "Aguardando pagamento",
    OrderStatus.Paid => "Pago, aguardando envio",
    OrderStatus.Shipped => "Enviado",
    OrderStatus.Cancelled => "Cancelado",
    _ => throw new ArgumentOutOfRangeException(nameof(status), status, "Status desconhecido")
};
```

O `_ => throw` no final não é defensivo demais — é o que garante que, se um novo valor de enum for adicionado, o código falhe ruidosamente em vez de silenciosamente cair num caso errado.

## Tipos que não permitem estado inválido

Prefira modelar regras de negócio no tipo em vez de validar depois. Um `record` com construtor validante, ou um "strongly-typed ID", evita que um `Guid` de cliente seja passado onde um `Guid` de pedido era esperado:

```csharp
public readonly record struct CustomerId(Guid Value)
{
    public static CustomerId New() => new(Guid.NewGuid());
}

public readonly record struct OrderId(Guid Value);
```

Isso parece verboso à primeira vista, mas elimina uma classe inteira de bugs de troca de parâmetros que só apareceriam em runtime (ou pior, em produção).

## Features modernas a preferir

- **Primary constructors** para reduzir boilerplate em classes simples (serviços, validators).
- **File-scoped namespaces** (`namespace MyApp.Orders;`) em vez de blocos com chaves — reduz um nível de indentação em todo arquivo.
- **Collection expressions** (`[1, 2, 3]`, `Span<int> s = [..]`) quando o target framework suportar.
- **Required members** (`required init`) para forçar inicialização de propriedades obrigatórias sem depender de construtor gigante.

## Anti-padrões a evitar (e por quê)

| Anti-padrão | Por que evitar |
|---|---|
| AutoMapper para tudo | Esconde o que está sendo mapeado; erros de mapeamento só aparecem em runtime, não em compile-time. Mapeamento explícito (construtor, `ToDto()`) é mais fácil de debugar. |
| Setters públicos livres em entidades de domínio | Permite que qualquer código mude o estado sem passar pela regra de negócio (ex.: mudar `Status` sem validar transição). |
| `catch (Exception) { }` silencioso | Esconde falhas reais; pelo menos logue e decida conscientemente se deve continuar ou relançar. |
| Lógica de negócio em controllers/endpoints | Dificulta testar sem subir o pipeline HTTP inteiro. Mova para um handler/serviço de aplicação (ver `architecture.md`). |
| `async void` fora de event handlers | Exceções não capturadas derrubam o processo; sempre use `async Task`. |
| Reflection pesada "por flexibilidade" | Custo de performance e de rastreabilidade — o IDE não consegue mais "ir para definição". Só vale quando o ganho é claro (ex.: serialização). |
