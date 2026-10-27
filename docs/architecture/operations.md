# Operação: Observabilidade, Logging, Configuração, CI/CD e Deploy

> Status: **Proposto** · Plataforma de hospedagem e backend de observabilidade são decisões pendentes (D-02, D-11); o desenho abaixo é agnóstico.

## 1. Observabilidade

**Padrão: OpenTelemetry** (traces, métricas, logs) exportado via OTLP para qualquer backend (D-11). Instrumentação automática: ASP.NET Core, HttpClient, EF Core/Npgsql, runtime; manual: `ActivitySource`/`Meter` por módulo (`Nexa.Tasks`, `Nexa.Notifications`, …).

### 1.1 Traces
- Contexto W3C propagado: SPA → Api → outbox (o `traceparent` é gravado na mensagem) → Worker → consumidor. Um trace conta a história "criar tarefa → lembrete agendado".
- Spans para: caso de uso (behavior), claim de lote, envio de push, chamada ao LLM (sem conteúdo), aprovação de sugestão.
- Amostragem: 100 % em baixo volume (app pessoal); revisar se custo exigir.

### 1.2 Métricas (mínimo)
| Área | Métrica |
|---|---|
| HTTP | latência/volume/erros por rota e status (RED) |
| Notificações | `reminders_due_backlog`, `reminder_dispatch_lag_seconds`, `push_send_total{outcome,status}`, `reminders_missed_total`, `reminders_failed_total`, `subscriptions_gone_total`, `dispatcher_heartbeat` |
| Outbox | `outbox_pending`, `outbox_oldest_age_seconds`, `outbox_failed_total` |
| IA | `ai_requests_total{outcome}`, `ai_latency_seconds`, `ai_tokens_total{direction}`, `ai_changes_discarded_total`, `ai_suggestion_decisions_total{kind,decision}` |
| Banco | pool, tempo de query, conexões, `deadlocks` |
| Auth | falhas de login, lockouts, `429` |

### 1.3 Health e alertas
- `/health/live` (processo vivo) e `/health/ready` (PostgreSQL acessível, migrations compatíveis). Worker expõe heartbeat próprio (métrica + health endpoint interno).
- **Alertas que importam** (poucos e acionáveis):
  1. `reminders_due_backlog > 0` por > 2 min **ou** `dispatcher_heartbeat` ausente (*dead-man switch* — lembrete parado é a falha mais cara).
  2. `outbox_oldest_age_seconds` > 5 min.
  3. Erros 5xx > limiar; readiness falhando.
  4. Push `401/403` (VAPID quebrado) ou taxa de falha > limiar.
  5. Falha de backup / PITR atrasado.
  6. Estouro de cota/custo de IA.
- SLO propostos (revisáveis): disponibilidade da API 99,5 %; p95 de despacho ≤ 30 s após `due_at`.

## 2. Logging

- **Serilog** → JSON estruturado no stdout (12-factor) + enriquecimento: `traceId`, `spanId`, `userId` (opaco), `module`, `requestId`, `env`, `version`. Logs também via OTLP.
- Níveis: `Information` para marcos de negócio e ciclo do Worker; `Warning` para retries/degradação; `Error` para falhas que exigem atenção; `Debug` desligado em produção.
- **Regras de conteúdo (obrigatórias)**: nunca logar senha, tokens, cookies, chaves, `endpoint`/chaves de push, e-mail em claro, **títulos/notas/descrições** ou prompts/respostas de IA. Use ids. Enriquecedor/destructuring policy bloqueia propriedades marcadas `[Sensitive]`; teste de arquitetura impede `ToString()` de entidades em logs.
- Mensagens em *templates* (`"Reminder {ReminderId} dispatched"`), nunca interpolação; códigos de evento estáveis para queries/alertas.
- Erros de Worker incluem causa classificada (transitória/permanente) e tentativa.
- Retenção curta (ex.: 14–30 dias) e acesso restrito.

## 3. Configuração por ambiente

| Item | Dev | Staging | Produção |
|---|---|---|---|
| Fonte de segredos | `dotnet user-secrets` / `.env` local (ignorado) | Secret store da plataforma | Secret store da plataforma |
| PostgreSQL | docker compose | Instância pequena gerenciada | Gerenciada + PITR + backups |
| Worker | Embutido na Api (`Workers:Enabled=true`) | Processo separado | Processo separado (≥1; ≥2 opcional) |
| LLM | `FakeLlmClient` por padrão | Provedor real com cota baixa | Provedor real |
| Push | VAPID de dev; subscription local | VAPID próprio | VAPID próprio |
| Registro de usuários | Habilitado | Desabilitado | **Desabilitado** (D-08) |
| Logs | Console legível | JSON | JSON + OTLP |

Regras: mesma imagem em todos os ambientes; options tipadas validadas no startup (`ValidateOnStart`); variáveis hierárquicas (`Ai__Provider`); nomes documentados em `.env.example` e `docs/config.md` (a criar na implementação); **nenhum segredo** em repositório, imagem ou frontend; configuração pública do SPA via `/config.json`.

## 4. CI/CD (GitHub Actions)

**Branching**: trunk-based; branches curtas; PR obrigatório; `main` protegida (checks obrigatórios, revisão, histórico linear).

### 4.1 Pipeline de PR (rápido, bloqueante)
1. Checkout, cache NuGet/npm; `dotnet restore --locked-mode`, `npm ci`.
2. **Backend**: build `-warnaserror`, `dotnet format --verify-no-changes`, testes unitários + arquitetura, testes de integração (Testcontainers), cobertura.
3. **Contrato**: gerar OpenAPI → `oasdiff` contra baseline; regenerar cliente TS → falhar se houver diff não commitado; snapshot de eventos.
4. **Frontend**: ESLint, `tsc`, Vitest, build, orçamento de bundle (3D em chunk separado).
5. **Segurança**: CodeQL (C# e JS/TS), gitleaks, `dotnet list package --vulnerable`, `npm audit`, dependency review, Trivy (imagem e IaC).
6. Build da imagem (sem push) para validar Dockerfile.

### 4.2 Pipeline de `main`
1. Tudo do PR + **E2E (Playwright)** contra ambiente efêmero (compose) + Lighthouse PWA.
2. Build e **push das imagens** (`nexa-api`/`nexa-worker` ou uma imagem com dois *entrypoints*; tag = SHA; assinadas/atestadas, SBOM anexado).
3. Gerar **migration bundle** (`dotnet ef migrations bundle`) como artefato.
4. Deploy automático em **staging** → smoke tests (health, login de teste, criar tarefa, lembrete de teste via fake gateway).
5. **Produção** com aprovação manual (GitHub Environment protection): aplicar migrations (bundle) → rollout Api → rollout Worker → smoke.

### 4.3 Princípios
- **Migrations em duas fases (expand/contract)**: durante o rollout coexistem versão antiga e nova do Api/Worker; toda migration deve ser compatível com a versão anterior do código. Remoção de coluna só uma release depois.
- **Rollback** = reimplantar a imagem anterior; não há `down` de migration em produção (por isso expand/contract).
- Autenticação do CI com a nuvem/plataforma via **OIDC** (sem chaves estáticas); `GITHUB_TOKEN` com permissões mínimas; actions fixadas por SHA.
- Versionamento semântico das imagens/releases com changelog gerado; Dependabot/Renovate semanal.
- Deploy do SPA e do SW: assets com hash imutáveis; `index.html` e `sw.js` com `no-cache`.

## 5. Estratégia de deploy

### 5.1 Topologia recomendada (agnóstica de provedor)
- **Imagem única de backend** (build multi-stage, usuário não-root, `readOnlyRootFilesystem`), com dois comandos: `Nexa.Api` e `Nexa.Worker`.
- **SPA**: na fase inicial empacotado e servido pela própria Api (`UseStaticFiles` + fallback SPA) → **uma origem, um artefato**, sem CORS. Separação para CDN/edge só se necessário, mantendo mesma origem por roteamento de caminho.
- **Edge/Reverse proxy** gerenciado: TLS, HSTS, HTTP/2-3, compressão; roteamento `/api` → Api.
- **PostgreSQL gerenciado** com backups automáticos, PITR, criptografia em repouso, versão suportada recente. Rede privada entre app e banco.
- **Segredos** no secret store; identidade de workload para acesso.
- Réplicas: Api ≥ 1 (stateless; Data Protection keys no banco); Worker ≥ 1 (≥ 2 opcional por segurança de disponibilidade — seguro por leases).

### 5.2 Rollout
- Rolling/blue-green simples com *readiness* gate. Worker: drenagem graciosa (`StopAsync` termina o lote em andamento; leases expiram se morrer — sem perda).
- Janelas: deploy não requer janela porque lembretes sobrevivem ao reinício (catch-up imediato; `MaxLateness` evita avisos velhos).
- Feature flags por configuração para liberar gradualmente (`Ai:Enabled`).

### 5.3 Backup, restauração e DR
- Backup diário + PITR; **teste de restauração trimestral** documentado; RPO alvo ≤ 5 min (PITR), RTO alvo ≤ 4 h (app pessoal) — revisáveis.
- Segredos e chaves VAPID/Data Protection: cópia segura offline (perder VAPID = todas as subscriptions precisam ser refeitas).

### 5.4 Ambientes locais
`docker compose up` sobe PostgreSQL, Api (com Worker embutido) e SPA em modo dev (Vite proxy `/api`). `FakeLlmClient` e gateway de push fake por padrão. Um comando para rodar migrações e *seed* mínimo de desenvolvimento.

## 6. Runbooks mínimos (a escrever junto da implementação)
Dispatcher parado / backlog de lembretes · rotação de VAPID · rotação de chave do LLM · restauração de banco · reprocessar mensagens do outbox com falha · desativar IA rapidamente (`Ai:Enabled=false`) · exclusão de usuário.
