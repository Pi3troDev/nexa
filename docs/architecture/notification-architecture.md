# Arquitetura de Notificações

> Status: **Proposto** · Decisão associada: [ADR-0004](../adr/0004-notification-strategy.md)

## 1. Princípios

1. **O backend é dono do agendamento.** Nenhum lembrete crítico depende de `setTimeout`, de a aba estar aberta ou de o SW estar vivo. O cliente apenas **registra a inscrição** e **exibe** o que chega.
2. **Persistente antes de tudo**: um lembrete só "existe" depois de gravado em PostgreSQL; sobrevive a restart, deploy e queda.
3. **At-least-once + colapso no dispositivo** (`tag`/`Topic` = `reminderId`) em vez de promessa de exactly-once.
4. **Entrega ao dispositivo é *best-effort*** (push services não garantem). "Confiável" aqui significa: o sistema **sempre tenta, registra o resultado, reintenta quando útil e oferece fallback in-app** — nunca perde silenciosamente.
5. Notifications é **módulo de suporte**: não conhece Tasks/Calendar/Habits; recebe pedidos genéricos.

## 2. Contrato de entrada (Notifications.Contracts)

```csharp
// Conceitual — assinatura final definida na implementação
public interface IReminderScheduler            // usada por Tasks/Calendar/Habits (via outbox)
{
    // Evento de integração equivalente: ReminderRequested / ReminderCancelled
}

public sealed record ReminderRequested(
    Guid EventId, Guid UserId,
    string SourceModule, string SourceType, Guid SourceId, string Kind,   // ex.: "tasks","TaskItem",id,"due-offset:-30m"
    DateTimeOffset? OccurrenceAt,            // identifica a ocorrência (recorrentes)
    DateTimeOffset DueAtUtc,                 // quando disparar
    DateTimeOffset ExpiresAtUtc,             // depois disso não faz mais sentido avisar
    string Title, string? Body, string DeepLink,
    long SourceVersion);                     // para ignorar eventos antigos

public sealed record ReminderCancelled(Guid EventId, Guid UserId,
    string SourceModule, string SourceType, Guid SourceId, string? Kind, long SourceVersion);
```

- Chave natural: `(UserId, SourceModule, SourceType, SourceId, Kind, OccurrenceAt)` — **UNIQUE**. Reenvio = upsert, não duplicata.
- `SourceVersion` ≤ versão já aplicada → ignorado (evento atrasado/duplicado). `ReminderCancelled` vence se versão ≥ atual; lembretes `Dispatched` não são "desfeitos" (já foram).
- Quem solicita **calcula** `DueAtUtc` (conhece a regra de negócio e o fuso); Notifications não reinterpreta regras de domínio alheias.

## 3. Modelo de dados (schema `notifications`)

| Tabela | Colunas principais | Índices/constraints |
|---|---|---|
| `reminders` | `id`, `user_id`, `source_*`, `occurrence_at`, `due_at`, `expires_at`, `status`, `attempts`, `lease_until`, `lease_token`, `source_version`, `content_*`, `version` | UNIQUE natural; **índice parcial** `(due_at) WHERE status IN ('Pending','Leased')` |
| `deliveries` | `id`, `reminder_id`, `subscription_id`, `channel`, `status`, `provider_status`, `attempted_at` | UNIQUE `(reminder_id, subscription_id, attempt_no)` |
| `push_subscriptions` | `id`, `user_id`, `endpoint_hash` UNIQUE, `endpoint`, `p256dh`, `auth` (protegidos), `label`, `status`, `last_success_at` | |
| `inbox_items` | `id`, `user_id`, `title`, `body`, `deep_link`, `created_at`, `read_at`, `reminder_id?` | `(user_id, created_at DESC)` |
| `outbox_messages` / `inbox_messages` | mecânica de eventos do módulo | PK `(consumer, message_id)` na inbox |

Estados de `Reminder`: `Pending → Leased → Dispatched | Pending (retry) | Failed`; `Pending|Leased → Cancelled`; `Pending → Missed | Expired`.

## 4. Despacho

**Worker `ReminderDispatcher`** (`BackgroundService`):

1. **Loop**: dorme `min(PollInterval=10s, tempo até o próximo due_at conhecido)`; também reage a um *nudge* in-process quando um lembrete próximo é criado (otimização opcional — correção não depende dela).
2. **Claim atômico** de lote (ex.: 50) com `FOR UPDATE SKIP LOCKED`, marcando `Leased`, `lease_until = now()+60s`, novo `lease_token`. Várias instâncias coexistem sem coordenação extra.
3. Para cada lembrete:
   - `now > expires_at` → `Expired` (+ inbox, se ainda útil).
   - `now > due_at + MaxLateness` (padrão 15 min, configurável por `Kind`) → `Missed` + **InboxItem** (o usuário vê "você perdeu…" ao abrir).
   - Senão: aplica **preferências** (Settings: canais habilitados, `QuietHours`). Em silêncio: adia para o fim da janela **se** ainda dentro de `expires_at`, senão inbox silencioso.
   - Envia para **cada** subscription ativa do usuário, em paralelo limitado, com timeout curto.
4. **Resultado por entrega**: `2xx` → sucesso; `404/410` → `Gone` (remove do envio futuro); `429` → respeitar `Retry-After`; `5xx/timeout` → transitório; `400/401/403/413` → permanente (VAPID/payload inválidos → alerta).
5. **Fechamento** (UPDATE condicional por `lease_token`): sucesso em ≥1 subscription → `Dispatched`; todas transitórias → volta `Pending` com `due_at = now + backoff` (30 s, 2 min, 10 min, 1 h; jitter) até `expires_at` ou `MaxAttempts=5` → `Failed` + inbox; zero subscriptions → somente inbox.
6. **Reaper**: lembretes `Leased` com lease expirada voltam a `Pending` (incrementando tentativa) — cobre crash do Worker.
7. **Alvo de latência (proposta, não requisito)**: p95 de `dispatch_time - due_at` ≤ 30 s com Worker saudável; medido por métrica (ver `operations.md`).

## 5. Janela de materialização (recorrentes)

Recorrência infinita não gera linhas infinitas. Um job periódico (protegido por `pg_try_advisory_lock`; idempotente por chave única) pede aos módulos de origem (via `Contracts`) as ocorrências dos próximos **N dias (padrão 14)** e emite `ReminderRequested` por ocorrência. Alterar/cancelar a regra de origem emite `ReminderCancelled` para as futuras e recria as novas. Mudança de fuso em Settings (`UserSettingsChanged`) dispara rematerialização.

## 6. Web Push — detalhes

- Protocolo: RFC 8030 (Web Push), **VAPID** RFC 8292, criptografia de mensagem RFC 8291 (`aes128gcm`). Biblioteca .NET madura atrás de `IWebPushGateway` (candidata: `Lib.Net.Http.WebPush`; validar manutenção na implementação). Nenhum outro código conhece a biblioteca.
- Chaves VAPID: par gerado uma vez por ambiente; **privada** no secret store; pública servida em `GET /api/v1/notifications/vapid-public-key`. Rotacionar invalida subscriptions (exigir re-subscribe).
- Cabeçalhos: `TTL` = tempo até `expires_at` (limitado), `Urgency` por `Kind`, `Topic` = `reminderId` (colapsa duplicatas no push service).
- Payload (cifrado): `{ reminderId, title, body?, deepLink }` — mínimo; "ocultar conteúdo" (D-14) troca por texto genérico.
- **Service Worker** (`push` event): **sempre** chama `showNotification` (Chrome exige `userVisibleOnly: true`; omitir pode gerar notificação genérica/revogação), com `tag=reminderId`, `data.deepLink`; `notificationclick` foca/abre a PWA no deep link e opcionalmente chama `POST /inbox/{id}/read`.
- Inscrição: `POST /api/v1/notifications/push-subscriptions` (idempotente por `endpoint_hash`); permissão pedida **após gesto** do usuário; `pushsubscriptionchange` re-registra. Logout remove a subscription do dispositivo.

### Limitações de plataforma (documentadas, não escondidas)

- **iOS/iPadOS**: Web Push funciona apenas para PWA **instalada na Tela de Início** (iOS 16.4+). A UI deve guiar a instalação antes de oferecer notificações.
- Navegadores podem atrasar entregas em economia de bateria; push services podem descartar após o `TTL`. Por isso: caixa de entrada in-app + reconciliação ao abrir o app (lista "perdidos").
- Usuário pode revogar a permissão: a UI detecta (`Notification.permission`) e mostra estado "notificações desligadas" com orientação — o servidor continua gerando inbox.

## 7. Reconciliação no cliente (rede de segurança)

Ao abrir/retomar a PWA (`visibilitychange`), o cliente chama `GET /notifications/inbox?since=…`. Itens `Missed`/não lidos aparecem. Isso é leitura do estado do servidor — **não** agendamento no cliente.

## 8. Observabilidade específica

Métricas: `reminders_due_backlog` (pendentes vencidos), `reminder_dispatch_lag_seconds` (histograma), `push_send_total{outcome,status}`, `reminders_missed_total`, `subscriptions_gone_total`, `dispatcher_heartbeat`. Alertas: backlog > 0 por > 2 min; heartbeat ausente; taxa de falha > limiar; `401/403` do push service (VAPID). Detalhes: [`operations.md`](operations.md).

## 9. Testes específicos

- Dispatcher com `FakeTimeProvider` e `IWebPushGateway` fake: sucesso, 410, 429, timeout, crash no meio (lease expira), duas instâncias concorrentes sem duplicar, lembrete vencido (`Missed`), expirado, silêncio.
- Idempotência: mesmo `ReminderRequested` N vezes → 1 linha; evento antigo ignorado; cancelar vence.
- Teste de integração real com PostgreSQL (Testcontainers) para `SKIP LOCKED` e índices parciais.
- E2E manual/semiautomatizado do caminho de push real em staging (browser real) — fora do gate de CI.

## 10. Evolução

Se a escala exigir, trocar o *claim por polling* por fila com mensagens atrasadas implementando as mesmas portas (`IReminderDispatcher`/`INotificationChannel`). O **banco continua sendo a fonte de verdade** dos lembretes; a fila vira apenas transporte.
