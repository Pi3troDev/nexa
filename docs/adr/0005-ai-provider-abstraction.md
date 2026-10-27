# ADR-0005: Abstração de provedor de IA e aprovação explícita

- Status: **Proposto**
- Data: 2026-10-26

## Context

O produto gera sugestões de planejamento com LLM. Requisitos: o domínio não pode depender de um fornecedor; a IA **não** controla a agenda — deve sugerir e pedir aprovação explícita antes de modificar dados. Dados do usuário são sensíveis; o conteúdo (títulos, notas) é não confiável e pode conter prompt injection. Provedores, modelos e preços mudam rápido.

## Decision

1. Módulo `AiPlanning` com **porta `ILlmClient`** em `Application` (saída estruturada por JSON Schema, tipos de erro neutros); **adapters** por provedor em `Infrastructure`; `FakeLlmClient` para dev/testes; seleção por configuração (`Ai:Provider`).
2. **Domínio de sugestões** próprio (`SuggestionRequest`, `Suggestion`, `ProposedChange`) — nenhum tipo do SDK do provedor cruza a fronteira.
3. **Propor ≠ executar**: o LLM e o runner só leem. A escrita ocorre somente no caso de uso `Approve`, **como o usuário**, via `Command API` dos módulos alvo, com `If-Match` na versão-base (mudanças desatualizadas viram `Stale`).
4. **Catálogo fechado** de tipos de mudança, sem operações destrutivas na v1; validação sintática (schema) e semântica (alvo existe/é do usuário/horizonte/conflitos). Saída inválida é descartada, não "consertada".
5. **Aprovação explícita por mudança**, com diff exato exibido; sugestões expiram; nada é aplicado automaticamente.
6. **Consentimento por usuário** (padrão desligado) e **minimização** de contexto por allowlist de campos.
7. Execução **assíncrona** no Worker (`202` + status), com timeout, resiliência (retry/circuit breaker), cotas por usuário e ledger de uso.
8. Templates de prompt **versionados** e independentes de provedor; logs sem prompts/respostas por padrão.
9. Nenhum teste automatizado chama LLM real.

## Alternatives considered

1. **Integrar diretamente o SDK de um fornecedor na Application/Domain**: mais rápido hoje, mas gera lock-in e contamina o domínio; rejeitado.
2. **Biblioteca de abstração de terceiros** (ex.: abstrações genéricas de chat/LLM): reduz código de adapter, mas impõe o modelo de dados da biblioteca e acopla ao seu ciclo de vida; pode ser usada *dentro* de um adapter, nunca como contrato do domínio.
3. **IA autônoma com ferramentas de escrita** (agente que altera a agenda): viola a regra de aprovação explícita; rejeitado.
4. **Aprovação "em lote único" apenas**: simples, mas esconde detalhes; preferimos granularidade por mudança (lote é atalho de UI).
5. **Chamada síncrona na requisição HTTP**: simples, porém a latência do LLM e os timeouts degradam a Api; rejeitado em favor de pedido assíncrono.
6. **Modelo local/auto-hospedado como padrão**: ótimo para privacidade, custo de operação/qualidade; permanece possível como mais um adapter.

## Consequences

**Positivas**: troca de provedor/modelo sem tocar domínio; segurança por construção (pior caso = proposta ruim visível ao usuário); auditoria completa da cadeia pedido → aprovação → aplicação; custo controlado; testes determinísticos.

**Negativas / custos**: mais código que uma chamada direta (porta, validação, estados); prompts e schemas precisam ser reavaliados a cada troca de modelo (a abstração não elimina diferenças de qualidade); denominador comum entre provedores limita recursos específicos (compensar com capacidades opcionais no adapter); UX de aprovação por mudança exige design cuidadoso; fluxo assíncrono requer polling/SSE na UI; escolha do primeiro provedor segue pendente (D-03).
