# Arquitetura de IA (AI Planning)

> Status: **Proposto** · Decisão associada: [ADR-0005](../adr/0005-ai-provider-abstraction.md)

## 1. Regras inegociáveis

1. **A IA não controla a agenda.** Ela analisa contexto, **propõe**, e **aguarda aprovação explícita** do usuário antes de qualquer alteração.
2. **Nenhuma escrita parte do LLM, do prompt ou do runner.** A escrita acontece somente no handler de `approve`, com a identidade do usuário e a mesma autorização das chamadas HTTP comuns.
3. **O domínio não conhece fornecedor de LLM.** Só `ILlmClient` (porta) e adapters na Infrastructure.
4. **Sem consentimento, sem chamada.** Dados só saem para o provedor se `Settings.AiConsent` estiver ativo (padrão: desligado).
5. **Conteúdo do usuário é dado não confiável** (risco de prompt injection) — nunca é tratado como instrução.

## 2. Estrutura

```
AiPlanning.Domain
  SuggestionRequest, Suggestion, ProposedChange (+ estados), AiUsageLedger
AiPlanning.Application
  Ports:   ILlmClient, IPromptTemplateRepository?, IContextAssembler, IChangeApplier, IAiUsageGuard
  Cases:   RequestSuggestion, RunSuggestion (job), GetSuggestion, ApproveChanges, RejectSuggestion
  Prompts: PromptTemplate (versionado, independente de provedor), OutputSchema (JSON Schema por Kind)
AiPlanning.Infrastructure
  Adapters: <Provider>LlmClient (um por provedor), FakeLlmClient (tests/dev), ResilienceHandlers
  EF: AiPlanningDbContext, repositórios
AiPlanning.Contracts
  IAiPlanningApi (se algum módulo precisar), eventos de auditoria
```

### 2.1 Porta de provedor (conceitual)

```csharp
public interface ILlmClient
{
    Task<LlmResult> GenerateStructuredAsync(LlmRequest request, CancellationToken ct);
}

public sealed record LlmRequest(
    string Purpose,                         // "day-plan", "week-plan" (métrica/roteamento)
    string SystemInstructions,              // confiável — vem do PromptTemplate
    IReadOnlyList<UntrustedContentPart> UserContext,   // dados do usuário, rotulados como não confiáveis
    JsonSchemaDefinition OutputSchema,      // saída estruturada obrigatória
    int MaxOutputTokens, TimeSpan Timeout, double? Temperature);

public sealed record LlmResult(
    string Json, LlmUsage Usage, string ProviderId, string ModelId, LlmFinishReason FinishReason);
```

Erros **neutros** de provedor (`LlmTransientError`, `LlmRateLimited`, `LlmRefused`, `LlmInvalidOutput`, `LlmTimeout`) — adapters traduzem os erros do SDK; a Application decide retry/falha sem saber quem é o provedor. Seleção do adapter por configuração (`Ai:Provider`); trocar de provedor = trocar registro de DI + config + reavaliar prompts.

## 3. Fluxo (resumo; sequência completa em [`data-flow.md`](data-flow.md) §4)

1. **Pedido** (`POST /ai-planning/suggestion-requests`) → valida consentimento, cotas (`AiUsageLedger`) e escopo; grava `Queued`; responde `202`. Idempotente.
2. **Runner no Worker** (claim com `SKIP LOCKED`): monta contexto **mínimo** por escopo via `Contracts` de leitura (Planning/Tasks/Calendar/Habits/Goals/Projects/Settings) → chama `ILlmClient` com timeout/resiliência → **valida**.
3. **Validação em duas etapas** (anti-corrupção):
   - *Sintática*: JSON Schema por `Kind`.
   - *Semântica*: catálogo fechado de `Kind`; alvo existe e é do usuário; datas dentro do horizonte; sem sobreposição com compromissos fixos; limites de quantidade; sem operações destrutivas (D-13). Mudanças inválidas são **descartadas** e contadas (métrica de qualidade), não "consertadas" silenciosamente.
4. **Persistência** da `Suggestion` (`Proposed`, com `ExpiresAt`) e `ProposedChange[]` com `BaseVersion` do alvo; uso de tokens no ledger.
5. **Revisão pelo usuário**: UI exibe racional e **diff exato por mudança** (antes → depois, onde cai no calendário). Aprovar/rejeitar **por mudança** ou em lote.
6. **Aprovação** (`POST …/approve`): registra `Decision=Approved` (usuário, horário) e só então aplica cada mudança via `Command API` do módulo alvo **com `If-Match = BaseVersion`**. Divergência → `Outcome=Stale` (não aplica; a UI oferece regenerar). Resultado parcial é válido e explícito.
7. **Auditoria**: `AuditableActionOccurred` para pedido, aprovação e cada aplicação (`ActorKind=AiApproval`).

## 4. Catálogo inicial de mudanças (proposta — D-13)

| Kind | Efeito (ao aprovar) | Módulo alvo |
|---|---|---|
| `AddPlanBlock` | Cria bloco num `DayPlan` | Planning |
| `MovePlanBlock` | Move bloco existente | Planning |
| `SetTaskDue` | Ajusta prazo de tarefa | Tasks |
| `CreateTask` | Cria tarefa sugerida | Tasks |

Fora do catálogo (v1): excluir qualquer coisa, alterar compromissos de Calendar, mexer em hábitos/metas/configurações. Expandir o catálogo exige revisão de segurança + ADR curto.

## 5. Segurança e privacidade

- **Minimização**: `IContextAssembler` por escopo define *allowlist* de campos (ex.: título, horário, duração, prioridade; **sem** notas livres por padrão). Escopos de dados ligados ao consentimento (`AiConsent.Scopes`).
- **Prompt injection**: separar instruções (confiáveis) de dados (`UntrustedContentPart`, delimitados e rotulados); instruções proíbem seguir ordens contidas nos dados; saída apenas JSON do schema; qualquer texto fora do schema é descartado. Mesmo que o modelo seja enganado, o pior efeito é uma **proposta ruim** que o usuário vê e pode rejeitar.
- **Sem ferramentas de escrita para o modelo.** Se tool-calling for usado, só ferramentas de leitura já restritas ao usuário/escopo.
- **Segredos**: chave do provedor apenas em secret store; nunca no contexto, logs ou front.
- **Logs**: por padrão **não** registrar prompt/resposta; registrar `PromptTemplateVersion`, `ContextFingerprint` (hash), tokens, latência, resultado. Modo de depuração com redação é opt-in e de curta duração.
- **Cotas e custo**: limite por usuário/dia (requisições e tokens), teto de contexto, `MaxOutputTokens`, timeout, circuit breaker; falha do provedor degrada para "sugestões indisponíveis", sem afetar o resto do app.

## 6. Qualidade e evolução do prompt

- Templates **versionados** (`PromptTemplateVersion` armazenada em cada request) e independentes de provedor; adapters só ajustam formato de transporte/structured output.
- **Conjunto de avaliação offline** (cenários sintéticos: dia cheio, conflitos, sem tempo livre, injeção em título) executado manualmente/CI não bloqueante; métricas: validade do schema, % de mudanças descartadas, conflitos introduzidos, aprovação pelo usuário.
- Métrica de produto: taxa de aprovação/rejeição por `Kind` (insumo para evoluir o catálogo).

## 7. Resiliência e estados de falha

| Falha | Comportamento |
|---|---|
| Provedor indisponível/timeout | Retry limitado com backoff; request `Failed` com motivo; UI mostra "tente novamente" |
| Saída inválida | 1 retry com instrução de correção; senão `Failed` (`LlmInvalidOutput`) |
| Contexto mudou entre geração e aprovação | `Stale` por mudança; regenerar |
| Sugestão não revisada | Expira (`ExpiresAt`, padrão 24 h) — nunca aplica sozinha |
| Aprovação duplicada | Idempotente (estado + `Idempotency-Key`) |
| Cota excedida | `429` com explicação; nada é enviado ao provedor |

## 8. Fora de escopo (não assumido)

Chat livre com a IA, execução autônoma, aprendizado/personalização com dados do usuário, RAG sobre notas, múltiplos agentes. Nenhum desses foi solicitado.
