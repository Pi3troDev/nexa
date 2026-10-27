# ADR-0001: Modular Monolith

- Status: **Proposto**
- Data: 2026-10-26

## Context

Nexa é um sistema pessoal de produtividade com 11 áreas funcionais (Identity, Tasks, Calendar, Planning, Habits, Notifications, AI Planning, Goals, Projects, Settings, Audit). Equipe pequena, um domínio coeso, necessidade de consistência transacional dentro de cada área, necessidade de confiabilidade nos lembretes e de baixo custo operacional. A escala esperada é baixa (um ou poucos usuários), mas o produto deve poder evoluir sem reescrita. O enunciado pede explicitamente evitar microsserviços "para parecer sofisticado".

## Decision

Adotar um **monolito modular** em .NET 10:

- Dois processos de **um mesmo código** (`Nexa.Api`, `Nexa.Worker`) sobre **um PostgreSQL**.
- **Módulos = bounded contexts**, cada um com 5 projetos (`Contracts`, `Domain`, `Application`, `Infrastructure`, `Api`) e **schema próprio**.
- Fronteiras impostas por **compilador** (referências de projeto) e **testes de arquitetura**: um módulo só enxerga `Contracts` de outro; grafo de módulos é um DAG.
- Comunicação: consultas/comandos in-process via interfaces de `Contracts`; efeitos assíncronos via **transactional outbox** + consumidores idempotentes. Sem transações entre módulos.
- Sem biblioteca de mediator; handlers explícitos com decorators.
- Pontos de substituição (fila, canal, LLM, relógio) definidos como portas desde já; nada além disso é abstraído preventivamente.

## Alternatives considered

1. **Microsserviços**: custo operacional (deploy, rede, observabilidade distribuída, consistência) sem benefício na escala; rejeitado.
2. **Monolito em camadas tradicional (sem módulos)**: mais simples no início, mas as 11 áreas tenderiam a acoplar via entidades/DbContext único; difícil evoluir/extrair depois; rejeitado.
3. **Monolito modular com um projeto por módulo (pastas + testes de arquitetura)**: menos `csproj`, mas regras só garantidas por teste e disciplina; considerado válido, mas escolhemos a imposição pelo compilador. Pode-se colapsar projetos de módulos finos futuramente via novo ADR.
4. **Vertical slices puras (sem camadas por módulo)**: boa ergonomia, mas o enunciado exige Clean Architecture/DIP explícitos e o domínio de Notifications/AI se beneficia de portas claras; rejeitado como estrutura principal (podemos organizar *dentro* de Application por caso de uso).
5. **Serverless/funções**: lembretes persistentes e outbox exigem processos de longa vida e transações; sem ganho; rejeitado.

## Consequences

**Positivas**: simplicidade operacional (2 processos, 1 banco); refatoração atômica; transações locais fortes; caminho natural de extração (ex.: Notifications) porque não há FKs/joins entre schemas e a interface já é `Contracts`; testabilidade alta.

**Negativas / custos**: ~55 projetos (mitigar: Central Package Management, `Directory.Build.props`, template de módulo); disciplina para manter `BuildingBlocks` mínimo e `Contracts` enxutos; eventual consistência entre módulos exige idempotência e tratamento de ordem; um banco compartilhado é SPOF e limite de escala (aceito na escala atual).

**Revisitar quando**: um módulo precisar escalar/implantar independentemente, ou a equipe/ritmo de deploy divergir por módulo.
