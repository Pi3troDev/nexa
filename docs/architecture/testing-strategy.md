# Estratégia de Testes

> Status: **Proposto** · Objetivo: confiança para evoluir rápido, com foco no que mais dói se quebrar (lembretes, isolamento por usuário, aprovação da IA, fusos/DST).

## 1. Pirâmide e prioridades

```
            ┌────────────┐  E2E (Playwright): poucos, caminhos críticos
            │   E2E      │
         ┌──┴────────────┴──┐  Integração: API+PG real (Testcontainers), Worker, contratos
         │   Integração     │
     ┌───┴──────────────────┴───┐  Unitários: Domain/Application puros, rápidos, muitos
     │        Unitários         │
     └──────────────────────────┘  + Testes de arquitetura (regras de dependência) em toda execução
```

Cobertura é consequência, não meta: gate no CI de **cobertura de ramos ≥ 80 % em `*.Domain`** e relatório (sem gate) nas demais; mutação (Stryker.NET) opcional periódico em Domain.

## 2. Backend

| Tipo | Escopo | Ferramentas / técnica |
|---|---|---|
| **Unitário de domínio** | Invariantes, máquinas de estado, VOs, `OccurrenceExpander`, regras de streak | xUnit; asserções com Shouldly ou AwesomeAssertions (evitar FluentAssertions ≥ 8 por licença); builders de objetos de teste; sem mocks no domínio |
| **Unitário de aplicação** | Handlers com portas falsas | Fakes escritos à mão preferidos a mocks; `FakeTimeProvider`; `Result<T>` verificado |
| **Arquitetura** | Regras de dependência (system-overview §7), DAG de módulos, `IUserOwned`, ausência de `DateTime.Now/UtcNow` e `new HttpClient`, `Contracts` sem tipos de Domain, sem referência a `Infrastructure` por `Application` | NetArchTest ou ArchUnitNET; roda em todo PR |
| **Integração de persistência** | Repositórios, migrations aplicadas do zero, constraints, índices parciais, `xmin`, query filter por usuário | **PostgreSQL real via Testcontainers** (nunca SQLite/InMemory — comportamento diverge: `SKIP LOCKED`, `timestamptz`, `xmin`, jsonb) |
| **Integração de API** | Pipeline completo: auth, validação, ProblemDetails, ETag/If-Match, `Idempotency-Key`, rate limit, status codes | `WebApplicationFactory` + PG real; usuário de teste autenticado por esquema de teste |
| **Isolamento entre usuários** | Para **cada** recurso: B não lê/edita/apaga dados de A; listas não vazam; `404` | Suíte parametrizada reutilizável (obrigatória por módulo) |
| **Worker / concorrência** | Dispatcher com 2+ instâncias simultâneas; lease expirada; outbox/inbox duplicados; ordem invertida; materializador com `advisory lock` | Testcontainers; `FakeTimeProvider`; gateways fake; execução paralela real |
| **Contratos HTTP** | OpenAPI estável | Geração + `oasdiff` vs. baseline; testes de snapshot (Verify) do schema de eventos |
| **Contratos de eventos** | Serialização compatível com versões anteriores | Snapshot de payloads por `schemaVersion` |
| **Segurança** | Endpoints sem auth → `401`; headers (CSP/HSTS); CSRF; limites de payload | Testes de integração + checagem automática de que todo endpoint tem política |
| **Tempo / fuso** | DST (transições primavera/outono), mudança de fuso do usuário, recorrência mensal no dia 31, virada de ano | Testes baseados em tabela com fusos IANA reais |

### Padrões de teste obrigatórios para áreas críticas

- **Notifications**: ver `notification-architecture.md` §9 (matriz de resultados de push, crash/lease, duplicidade, silêncio, expiração).
- **AI Planning**: `FakeLlmClient` determinístico para todos os testes automatizados (**nenhum teste automatizado chama LLM real**). Casos: saída inválida; mudança fora do catálogo; alvo de outro usuário; **nenhuma escrita sem `Approved`** (teste de propriedade: percorrer todos os caminhos e afirmar que comandos de módulo alvo só são chamados após aprovação); `Stale`; expiração; aprovação duplicada; sem consentimento → provedor nunca chamado; injeção em título não altera o catálogo/efeitos.
- **Idempotência**: mesma `Idempotency-Key` ×N → um efeito; corpo diferente → `422`.

## 3. Frontend

| Tipo | Ferramentas |
|---|---|
| Unidade/componente | Vitest + Testing Library; consultas por papel/label (acessibilidade como efeito colateral) |
| Dados de rede | MSW (mock da API a partir do OpenAPI); cliente gerado garante tipos |
| Acessibilidade | `axe` nos testes de componente e no E2E |
| E2E | Playwright (Chromium + WebKit + Firefox nos caminhos críticos): login, criar tarefa, plano do dia, check-in de hábito, fluxo de aprovação da IA com backend de teste e `FakeLlmClient`, instalação/PWA smoke |
| PWA | Lighthouse CI (installable, SW registrado, orçamento de performance); teste de que o SW **não** cacheia `/api` autenticada |
| Push | Em CI: unidade do handler do SW (eventos `push`/`notificationclick` simulados — sempre chama `showNotification`). Entrega real: verificação manual/semiautomática em staging (não é gate) |
| 3D | Teste de fumaça: a cena monta com `@react-three/test-renderer`/headless sem lançar; **fallback 2D** renderiza se WebGL indisponível; view-models testados como funções puras. Sem snapshots visuais como gate no início |
| Tipos | `tsc --noEmit` strict; ESLint (incl. regra de import boundaries: `office3d` não importa `api/`/`features/`; features não se importam) |

## 4. Dados de teste, tempo e determinismo

- Nada usa relógio real: `TimeProvider` / `FakeTimeProvider` no backend; relógio simulado no frontend (`vi.useFakeTimers`).
- Dados: builders/Object Mothers; sem dumps; IDs determinísticos quando relevante.
- Testcontainers com um container por execução de suíte e *reset* rápido (Respawn) entre testes.
- Testes independentes, paralelizáveis, sem ordem implícita.

## 5. Gates por etapa (ver `operations.md` §4)

| Etapa | Obrigatório para merge |
|---|---|
| PR | Build com warnings como erro; unitários; arquitetura; integração (Testcontainers); contrato OpenAPI/eventos; frontend lint+typecheck+unit+build; scans de segurança |
| Main | Tudo acima + E2E em ambiente efêmero + Lighthouse PWA |
| Pré-produção | Smoke pós-deploy em staging; verificação manual de push real quando mudar Notifications/SW |

## 6. Não-funcionais

- **Carga leve** (k6) para endpoints de leitura agregada (`/planning/overview`) e para o dispatcher com backlog sintético (ex.: 10 mil lembretes vencidos) — verificar drenagem e ausência de duplicidade. Não é gate inicial.
- **Caos básico**: derrubar o Worker durante despacho; derrubar PG por segundos; gateway de push devolvendo 5xx/timeout — verificar recuperação sem perda/duplicidade.
- **Restauração de backup** exercitada periodicamente (runbook em `operations.md`).
