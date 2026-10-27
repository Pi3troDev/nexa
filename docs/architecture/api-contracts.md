# Contratos da API e Versionamento

> Status: **Proposto** · Os endpoints abaixo definem **recursos e semântica**, não implementação. Fonte de verdade final: OpenAPI gerado do código e versionado em `docs/api/openapi.v1.json` (verificado no CI).

## 1. Convenções gerais

| Tema | Regra |
|---|---|
| Base | `/api/v1/...`, JSON (`application/json`), UTF-8, nomes `camelCase` |
| Autenticação | Cookie de sessão (ver `security.md`); mutações exigem header antiforgery |
| Ids | UUID (v7) em string |
| Tempo | Instantes: ISO 8601 UTC (`2026-10-26T14:00:00Z`). Horário local de agenda: `{ "local": "2026-10-26T09:00:00", "timeZone": "America/Sao_Paulo" }`. Dia inteiro/datas: `yyyy-MM-dd` |
| Concorrência | Respostas de recurso trazem `ETag`; `PUT/PATCH/DELETE` exigem `If-Match` (`428` se ausente, `412` se divergente) |
| Idempotência | `Idempotency-Key` em `POST` com efeito (ver system-overview §11) |
| Paginação | Cursor: `?limit=50&cursor=…` → `{ items, nextCursor }`; `limit` máximo 200 |
| Filtros/ordenação | Query params explícitos e documentados por recurso; sem linguagem de consulta genérica |
| Criação | `201` + `Location` + corpo do recurso; operações assíncronas: `202` + `Location` do recurso de status |
| Remoção | `204`; idempotente (`DELETE` repetido → `204`/`404` documentado por recurso) |
| Erros | RFC 9457 `application/problem+json` (abaixo) |
| Correlação | Aceita/propaga `traceparent` (W3C); resposta inclui `traceId` nos erros |
| Cache | API autenticada: `Cache-Control: no-store` por padrão; `ETag` para revalidação condicional (`If-None-Match` → `304`) em leituras agregadas selecionadas |
| Rate limit | `429` + `Retry-After` |

### Formato de erro

```json
{
  "type": "https://docs.nexa.app/errors/tasks.title_required",
  "title": "Título é obrigatório",
  "status": 400,
  "code": "tasks.title_required",
  "traceId": "00-4bf92f35...-01",
  "errors": { "title": ["required"] }
}
```

`code` é estável e faz parte do contrato (o frontend ramifica por ele, nunca por `title`/mensagem). Catálogo mantido junto a cada módulo (`<Module>.Contracts/Errors`). *(A URL de `type` é ilustrativa.)*

## 2. Recursos por módulo (v1)

### Identity — `/auth`, `/me`
| Método | Rota | Notas |
|---|---|---|
| POST | `/auth/login` | Credencial/passkey → cookie. Rate limit estrito |
| POST | `/auth/logout` | Encerra sessão |
| POST | `/auth/passkeys/options` · `/auth/passkeys` | Registro/uso de passkey (WebAuthn) |
| POST | `/auth/mfa/totp/enable` · `/verify` | TOTP |
| GET | `/me` | Perfil mínimo do usuário atual |
| DELETE | `/me` | Exclusão de conta (reautenticação) |
| POST | `/auth/register` | **Desabilitado por padrão** (D-08) |

### Settings — `/settings`
`GET /settings` · `PUT /settings` (ETag/If-Match). Inclui fuso, locale, semana, silêncio, janelas de trabalho, preferências de notificação, consentimento de IA, preferências 3D.

### Projects — `/projects`, `/areas`
CRUD de `projects` (`GET` lista com filtro `status`, `areaId`; `POST`; `GET/PATCH/DELETE /{id}`), `POST /projects/{id}/archive`. `areas`: CRUD simples (D-05).

### Goals — `/goals`
CRUD; `POST /goals/{id}/milestones`, `PATCH /goals/{id}/milestones/{mid}`, `PUT /goals/{id}/progress`.

### Tasks — `/tasks`
| Método | Rota | Notas |
|---|---|---|
| GET | `/tasks?status&projectId&dueFrom&dueTo&q&cursor` | Lista |
| POST | `/tasks` | Corpo inclui `reminders: [{ "offset": "-PT30M" }]` |
| GET/PATCH/DELETE | `/tasks/{id}` | `If-Match` |
| POST | `/tasks/{id}/complete` · `/reopen` | Idempotentes |

### Calendar — `/calendar`
| Método | Rota | Notas |
|---|---|---|
| GET/POST | `/calendar/events` | Regra mestre do compromisso |
| GET/PATCH/DELETE | `/calendar/events/{id}` | `If-Match` |
| PUT/DELETE | `/calendar/events/{id}/exceptions/{occurrenceKey}` | Altera/cancela uma ocorrência |
| GET | `/calendar/occurrences?from&to` | Ocorrências **expandidas** (intervalo máx. configurável) |

### Habits — `/habits`, `/routines`
| Método | Rota | Notas |
|---|---|---|
| CRUD | `/habits`, `/habits/{id}` | `If-Match` |
| PUT/DELETE | `/habits/{id}/check-ins/{yyyy-MM-dd}` | Idempotente por chave natural |
| GET | `/habits/{id}/check-ins?from&to` · `/habits/{id}/stats` | Sequência/aderência derivadas |
| CRUD | `/routines`, `/routines/{id}` | `kind=sleep` para rotina de sono |

### Planning — `/planning`
| Método | Rota | Notas |
|---|---|---|
| GET | `/planning/days/{date}` | Plano do dia + blocos |
| PUT | `/planning/days/{date}` | Substitui blocos (`If-Match`); valida sobreposição |
| GET/PUT | `/planning/weeks/{isoWeek}` | Plano semanal |
| GET | `/planning/overview?date=` | **Leitura composta** "hoje" (Tasks+Calendar+Habits+Plano) — usada pela home e pelo 3D |
| GET | `/planning/week-overview?isoWeek=` | Idem para a semana |

### Notifications — `/notifications`
| Método | Rota | Notas |
|---|---|---|
| GET | `/notifications/vapid-public-key` | Chave pública |
| POST | `/notifications/push-subscriptions` | Idempotente por endpoint |
| DELETE | `/notifications/push-subscriptions/{id}` | Logout/revogação |
| GET | `/notifications/inbox?since&cursor` | Caixa in-app / perdidos |
| POST | `/notifications/inbox/{id}/read` · `/notifications/inbox/read-all` | |

Não há `POST /reminders` público: lembretes nascem dos recursos de origem (`reminders` no corpo de tarefas/compromissos/hábitos) para manter uma única fonte de verdade.

### AI Planning — `/ai-planning`
| Método | Rota | Notas |
|---|---|---|
| POST | `/ai-planning/suggestion-requests` | `202` + `Location`. Requer consentimento e cota |
| GET | `/ai-planning/suggestion-requests/{id}` | Status; quando `Completed`, inclui `suggestionId` |
| GET | `/ai-planning/suggestions/{id}` | Racional + mudanças + diffs |
| POST | `/ai-planning/suggestions/{id}/approve` | Corpo `{ "changeIds": [...] }`; **único** caminho que aplica |
| POST | `/ai-planning/suggestions/{id}/reject` | Corpo `{ "changeIds": [...]? }` |
| GET | `/ai-planning/usage` | Cota do dia |

### Audit — `/audit`
`GET /audit/entries?from&to&module&cursor` — somente leitura, do próprio usuário.

### Operacional (sem versão, sem autenticação de usuário)
`GET /health/live` · `GET /health/ready` · `GET /config.json` (apenas configuração pública).

## 3. Versionamento

1. **Versão major no caminho** (`/api/v1`). Explícita, cacheável, simples para PWA (clientes antigos permanecem no ar por dias/semanas por causa do SW).
2. **Dentro de uma major, somente mudanças compatíveis**: novos campos opcionais, novos endpoints, novos valores de enum *somente se* clientes foram escritos para tolerar desconhecidos (regra: o cliente trata enum desconhecido como "outro"). Proibido: remover/renomear campo, mudar tipo/semântica, tornar opcional → obrigatório.
3. **Quebra** ⇒ nova major (`/api/v2`) coexistindo com a anterior por uma janela definida; v1 marcada com `Deprecation` e `Sunset` (RFC 8594) e métrica de uso. Para uma app pessoal a janela pode ser curta, mas o mecanismo existe.
4. **Eventos de integração** seguem a mesma regra (aditivo) com `schemaVersion`; mudanças incompatíveis = novo tipo de evento publicado em paralelo até migrar consumidores.
5. **Gate de CI**: o OpenAPI gerado é comparado ao baseline (`oasdiff`); mudança incompatível em v1 **falha o build**. O cliente TypeScript é regenerado no CI e diferenças não commitadas falham o build.
6. Cabeçalho `X-Client-Version` (build do SPA) enviado pelo cliente apenas para diagnóstico; o servidor pode responder `426` se um dia precisar forçar atualização (não usado na v1).

## 4. Contratos internos entre módulos (`*.Contracts`)

Não são HTTP. Interfaces em C# + DTOs/eventos imutáveis (`record`), sem tipos de domínio. Exemplos: `ITasksReadApi`, `ITasksCommandApi`, `ICalendarReadApi`, `IHabitsReadApi`, `IUserPreferencesApi`, `IReminderScheduler`, eventos `ReminderRequested`, `UserDeleted`, `AuditableActionOccurred`. Regras: mudanças aditivas; alterações incompatíveis exigem revisão dos consumidores no mesmo PR (monolito = refatoração atômica) **mas** eventos persistidos no outbox seguem a regra de versionamento acima.
