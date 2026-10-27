# ADR-0004: Estratégia de notificações e lembretes

- Status: **Proposto**
- Data: 2026-10-26

## Context

Lembretes são funcionalidade crítica. O enunciado exige Web Push, agendamento persistente e confiável, e proíbe depender de `setTimeout` no frontend. O backend deve processar os lembretes. Hospedagem inicial com .NET Worker/Hosted Services, com possibilidade de migrar para fila dedicada. Plataformas têm limitações (iOS exige PWA instalada; push é best-effort).

## Decision

1. **Módulo `Notifications`** dono de `Reminder`, `Delivery`, `PushSubscription`, `InboxItem`. Outros módulos **solicitam** lembretes por contrato (`ReminderRequested/Cancelled` via outbox); Notifications não conhece Tasks/Calendar/Habits.
2. **Persistência primeiro**: lembrete gravado no PostgreSQL com `due_at`, `expires_at`, status e lease; chave natural única por origem/ocorrência (idempotência).
3. **Despacho no Worker** por polling com `FOR UPDATE SKIP LOCKED` + lease; seguro com múltiplas instâncias; reaper para leases expirados; retry com backoff/jitter até `expires_at`/`MaxAttempts`; política de atraso máximo (`MaxLateness` → `Missed`).
4. **Web Push** (RFC 8030/8291/8292, VAPID) atrás de `IWebPushGateway`/`INotificationChannel`; `tag`/`Topic` = `reminderId`; SW sempre exibe a notificação.
5. **Fallback in-app**: caixa de entrada persistida e reconciliação ao abrir o app; nada depende de timers do cliente.
6. **Recorrência** materializada em horizonte rolante (padrão 14 dias) por job idempotente.
7. Respeita preferências/silêncio de `Settings` (via `Contracts`).
8. **Portas de substituição** (`IReminderDispatcher`, `IOutboxProcessor`, `INotificationChannel`) permitem trocar polling por fila dedicada sem alterar domínio; o banco segue como fonte de verdade.

## Alternatives considered

1. **Timers no cliente (`setTimeout`/Notification Triggers/alarms de SW)**: não confiáveis (aba fechada, SW encerrado, suspensão do SO); explicitamente proibido para lembretes críticos.
2. **Fila/broker com mensagens atrasadas desde o início** (RabbitMQ, SQS, Service Bus, Hangfire/Quartz com store): bom em escala, mas mais um componente e risco de divergência entre o estado de negócio e a fila (dual-write); polling indexado no PG resolve a escala atual de forma transacional. Mantido como evolução.
3. **Hangfire/Quartz.NET** para agendar: adicionam modelo e schema próprios; o conceito de "lembrete" já é domínio com status, expiração, inbox; manter nosso modelo evita impedância. Rejeitado por ora.
4. **Notificações por e-mail/SMS como canal principal**: não solicitado; custo/privacidade; deixado atrás de `INotificationChannel` (D-10).
5. **PostgreSQL LISTEN/NOTIFY para acordar o Worker**: possível como otimização; não é necessário para correção; adiado.
6. **Push pelo próprio FCM/APNs SDK**: Web Push padrão é suficiente e independente de loja/plataforma.

## Consequences

**Positivas**: sobrevive a deploys, quedas e reinícios; sem perda silenciosa (todo desfecho é registrado: `Dispatched/Failed/Missed/Expired/Cancelled`); idempotente; escalável horizontalmente no Worker; testável com relógio e gateway falsos; independente do estado do cliente.

**Negativas / custos**: latência de entrega depende do intervalo de polling (alvo p95 ≤ 30 s — não é tempo real); entrega ao dispositivo é best-effort e limitada por plataforma (iOS exige instalação); polling no PG tem teto de escala; rotação de VAPID invalida subscriptions; materialização de horizonte adiciona um job a operar; necessidade de alertas de *dead-man switch* para detectar Worker parado.
