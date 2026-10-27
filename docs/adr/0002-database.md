# ADR-0002: Banco de dados e estratégia de persistência

- Status: **Proposto**
- Data: 2026-10-26

## Context

Precisamos de persistência transacional confiável para dados pessoais, para o agendamento de lembretes (com claim concorrente de workers), para outbox/inbox e para consultas por intervalo de tempo. O stack definido é C#/EF Core/PostgreSQL. Há necessidade de manter módulos desacoplados no nível de dados e de tratar fusos e recorrência corretamente.

## Decision

1. **PostgreSQL** como único banco (gerenciado, com backups e PITR).
2. **Schema por módulo**, **um `DbContext` por módulo**, migrations por módulo (bundle aplicado no deploy; não no startup). Sem FKs, JOINs ou views entre schemas.
3. **EF Core + Npgsql**; `snake_case`; ids **UUIDv7** gerados na aplicação; `timestamptz` UTC para instantes; `date` para dias; horário local + `TimeZoneId` (IANA) quando a semântica é "horário de parede" (recorrência, hábitos).
4. **Concorrência otimista** com `xmin` como row version → ETag/If-Match na API.
5. **Isolamento por usuário**: `user_id` em toda tabela de dados, `IUserOwned` + global query filter + índices com `user_id` à frente; teste de arquitetura. RLS fica como avaliação futura (D-09).
6. **Fila/lembretes no próprio PostgreSQL**: `FOR UPDATE SKIP LOCKED` + lease; outbox/inbox por módulo; locks consultivos (`pg_try_advisory_lock`) para jobs singleton.
7. `jsonb` apenas para payloads de mensagens e dados verdadeiramente semiestruturados; dados de domínio consultáveis são colunas.
8. Sem soft delete genérico; exclusão real para atender remoção de dados; estados de negócio explícitos quando o domínio pede.
9. Testes de integração sempre com PostgreSQL real (Testcontainers).

## Alternatives considered

1. **SQL Server**: ótimo com .NET, mas o enunciado define PostgreSQL; sem ganho que justifique o custo/licença. Rejeitado.
2. **SQLite**: simples para app pessoal, porém concorrência de escrita, `SKIP LOCKED`, tipos de tempo e multi-instância do Worker ficam limitados; rejeitado.
3. **Banco por módulo (instâncias separadas)**: isolamento máximo, operação cara demais agora; schema por módulo entrega a fronteira lógica com opção de separar depois.
4. **Banco NoSQL/documental para parte do domínio**: sem necessidade comprovada; rejeitado.
5. **Fila/broker dedicado desde o início** para lembretes/eventos: mais um componente crítico; outbox + polling no PG atende a escala e mantém atomicidade com a escrita de negócio. Substituível (ADR-0004).
6. **Event sourcing**: complexidade sem requisito; rejeitado. Auditoria cobre a necessidade de histórico.
7. **Um único `DbContext` global**: facilita JOINs, mas destrói fronteiras; rejeitado.

## Consequences

**Positivas**: atomicidade entre mudança de negócio e outbox; modelo operacional simples; extração futura de módulos facilitada; comportamento idêntico entre testes e produção.

**Negativas / custos**: sem JOINs entre módulos → leituras compostas via `Contracts` (custo de N+1 controlado por APIs de leitura em lote); migrations por módulo exigem disciplina de ordem/compatibilidade (expand/contract); polling no PG tem teto de escala (suficiente aqui); um banco é SPOF (mitigado por backups/PITR e provedor gerenciado); fusos/recorrência exigem NodaTime e testes dedicados.
