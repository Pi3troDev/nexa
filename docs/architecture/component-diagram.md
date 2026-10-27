# Diagramas de Componentes

> Status: **Proposto** · Diagramas em Mermaid (renderizam no GitHub).

## 1. Contêineres e infraestrutura

```mermaid
flowchart TB
  User((Usuário))
  subgraph Device["Dispositivo"]
    SPA["SPA React/TS\n(shell + features + office3d lazy)"]
    SW["Service Worker"]
  end
  subgraph Platform["Plataforma de execução"]
    Proxy["Reverse proxy / Edge\nTLS · HSTS · mesma origem"]
    subgraph Api["Nexa.Api (stateless, N réplicas)"]
      MW["Middleware: ProblemDetails, rate limit,\nauth, correlação, idempotência"]
      Mods1["Módulos (Api + Application)"]
    end
    subgraph Wrk["Nexa.Worker (1..N, seguro por lease)"]
      OBX["Outbox processors"]
      REM["Reminder dispatcher"]
      MAT["Materializador de horizonte"]
      AIJ["AI suggestion runner"]
      HK["Housekeeping (retenção, idempotency keys)"]
    end
    PG[("PostgreSQL\nschemas: identity, settings, audit,\nnotifications, projects, goals, tasks,\ncalendar, habits, planning, ai_planning")]
  end
  Push["Push services"]
  LLM["Provedor LLM"]
  Otel["OTLP backend"]

  User --> SPA
  SPA <--> SW
  SPA -->|"HTTPS /api/v1"| Proxy --> MW --> Mods1 --> PG
  OBX --> PG
  REM --> PG
  MAT --> PG
  AIJ --> PG
  HK --> PG
  REM -->|"Web Push (VAPID)"| Push --> SW
  AIJ -->|"ILlmClient adapter"| LLM
  Api -.-> Otel
  Wrk -.-> Otel
```

## 2. Camadas dentro de um módulo

```mermaid
flowchart LR
  subgraph Module["Módulo Tasks (exemplo)"]
    direction TB
    A["Tasks.Api\nEndpoints · DTOs HTTP · AddTasksModule()"]
    B["Tasks.Application\nCommands/Queries/Handlers · Validators\nPortas: ITaskRepository, IUnitOfWork"]
    C["Tasks.Domain\nTaskItem (AR) · VOs · Domain events · regras"]
    D["Tasks.Infrastructure\nTasksDbContext · repositórios · outbox handlers"]
    E["Tasks.Contracts\nITasksReadApi · ITasksCommandApi · IEs"]
  end
  A --> B --> C
  B --> E
  D --> B
  D --> C
```

## 3. Pipeline de um caso de uso (backend)

```mermaid
flowchart LR
  Req["HTTP"] --> EP["Endpoint\n(DTO → Command)"]
  EP --> Beh["Behaviors (decorators):\nlogging/trace → validação →\nautorização de recurso → unit of work"]
  Beh --> H["Handler (Application)"]
  H --> Dom["Agregado (Domain)"]
  H --> Repo["IRepository (porta)"]
  Repo -.-> Infra["EF Core / Npgsql"]
  H --> Out["IOutbox (mesma transação)"]
  Beh --> Res["Result<T> → Problem Details / 2xx"]
```

Sem biblioteca de mediator: `ICommandHandler<TCommand,TResult>` + decorators registrados explicitamente (composição simples, depurável, sem dependência comercial).

## 4. Worker — componentes e portas

```mermaid
flowchart TB
  subgraph Worker["Nexa.Worker"]
    direction TB
    S["Scheduler loop (BackgroundService)\npoll + sleep até próximo due"]
    S --> C1["Claim batch\nFOR UPDATE SKIP LOCKED + lease"]
    C1 --> D1["IReminderDispatcher"]
    D1 --> CH["INotificationChannel\n(WebPushChannel)"]
    CH --> GW["IWebPushGateway\n(adapter da lib de Web Push)"]
    D1 --> Rt["RetryPolicy\n(backoff + expiração)"]
    O["OutboxProcessor por módulo"] --> H2["Consumidores idempotentes (inbox)"]
  end
```

Substituição futura por fila dedicada: troca-se a implementação de `IOutboxProcessor`/`IReminderDispatcher` (ou apenas o *transporte* entre elas) sem tocar em Domain/Application.

## 5. Frontend

```mermaid
flowchart TB
  subgraph SPA["SPA"]
    Shell["app/ (router, providers, boundaries)"]
    Feat["features/* (UI + hooks de dados)"]
    Query["TanStack Query\n(cache de servidor)"]
    ApiC["api/ (cliente gerado do OpenAPI)"]
    VM["view-models (puros)"]
    O3D["office3d/ (R3F + Drei, lazy)"]
    PWA["pwa/ (install, push state)"]
  end
  Shell --> Feat --> Query --> ApiC
  Feat --> VM --> O3D
  PWA --> SWk["sw/ (Workbox)"]
  ApiC -->|"cookie HttpOnly\nCSRF header"| BE["/api/v1"]
```

Regras: `office3d` **não** importa `api/` nem `features/*` — recebe view-models por props/contexto (testável sem WebGL, trocável por 2D). Features não importam umas às outras; compartilham via `shared/`.

## 6. Pontos de extensão previstos

| Ponto | Hoje | Amanhã (se a escala/produto exigir) |
|---|---|---|
| Transporte de eventos | Outbox em PG + polling | Broker/fila |
| Despacho atrasado de lembretes | Polling indexado por `due_at` | Fila com delay |
| Canais | Web Push + in-app | Outros canais atrás de `INotificationChannel` |
| LLM | Um adapter configurado | Múltiplos/roteamento por tarefa |
| Módulo → serviço | In-process | Extração de um módulo (Notifications é o candidato natural) mantendo `Contracts` |
