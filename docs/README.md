# Documentação de Arquitetura — Nexa

## Índice

| Documento | Conteúdo |
|---|---|
| [architecture/system-overview.md](architecture/system-overview.md) | Visão geral, estrutura da solution, dependências, persistência, concorrência, idempotência, erros, configuração, revisão |
| [architecture/context-map.md](architecture/context-map.md) | Módulos/bounded contexts, relações, modelo de domínio inicial |
| [architecture/component-diagram.md](architecture/component-diagram.md) | Diagramas de contêineres, componentes, camadas, frontend |
| [architecture/data-flow.md](architecture/data-flow.md) | Fluxos principais (sequências) |
| [architecture/api-contracts.md](architecture/api-contracts.md) | Convenções da API, contratos por módulo, versionamento |
| [architecture/security.md](architecture/security.md) | Autenticação, autorização, ameaças, controles |
| [architecture/notification-architecture.md](architecture/notification-architecture.md) | Lembretes persistentes e Web Push |
| [architecture/ai-architecture.md](architecture/ai-architecture.md) | Provider abstraction e fluxo de aprovação |
| [architecture/testing-strategy.md](architecture/testing-strategy.md) | Pirâmide, ferramentas, gates |
| [architecture/operations.md](architecture/operations.md) | Observabilidade, logging, configuração, CI/CD, deploy |
| [adr/](adr/) | ADRs 0001–0006 |

Os arquivos `api-contracts.md` e `operations.md` foram adicionados além da lista mínima solicitada porque os itens 5, 10, 16–20 do escopo não cabiam em nenhum dos arquivos pedidos sem diluí-los.

## Onde cada item do escopo está

| # | Item | Documento |
|---|---|---|
| 1 | Contextos/módulos | context-map |
| 2 | Estrutura das soluções | system-overview §6 |
| 3 | Dependências entre camadas | system-overview §7–8 |
| 4 | Modelo de domínio inicial | context-map §4 |
| 5 | Contratos da API | api-contracts |
| 6 | Autenticação/autorização | security §2–3, ADR-0003 |
| 7 | Notificações | notification-architecture, ADR-0004 |
| 8 | IA | ai-architecture, ADR-0005 |
| 9 | Testes | testing-strategy |
| 10 | Observabilidade | operations §1 |
| 11 | Segurança | security |
| 12 | Persistência | system-overview §9, ADR-0002 |
| 13 | Concorrência | system-overview §10 |
| 14 | Idempotência | system-overview §11 |
| 15 | Tratamento de erros | system-overview §12 |
| 16 | Logging | operations §2 |
| 17 | Versionamento da API | api-contracts §3 |
| 18 | Configuração por ambiente | system-overview §13, operations §3 |
| 19 | CI/CD | operations §4 |
| 20 | Deploy | operations §5 |

## Decisões pendentes

| ID | Decisão | Recomendação provisória |
|---|---|---|
| D-02 | Plataforma de hospedagem e PostgreSQL gerenciado | Contêineres + PostgreSQL gerenciado com PITR; qualquer PaaS/nuvem que sirva |
| D-03 | Primeiro provedor de LLM e modelo | Escolher por custo/qualidade em avaliação offline; o domínio não muda |
| D-04 | Recorrência: RRULE (RFC 5545) completo vs. subconjunto | Subconjunto próprio (diária/semanal/mensal + intervalo + exceções) serializável como RRULE |
| D-05 | "Estudos" e "Trabalho": Áreas (classificação) em Projects? | Sim: `Area` pertence a Projects; Tasks referencia por id |
| D-06 | Progresso de Goals: manual vs. derivado de tarefas | Manual + marcos na v1 |
| D-07 | Escrita offline (fila de mutações) no PWA | Fora da v1: offline = shell + leitura do último estado; `Idempotency-Key` já viabiliza depois |
| D-08 | Registro aberto vs. convite/único usuário | Registro desabilitado por padrão (flag); dados já isolados por `user_id` |
| D-09 | PostgreSQL RLS como defesa em profundidade | Avaliar após v1 |
| D-10 | Canal de fallback além de push/in-app (e-mail, etc.) | Não implementar sem requisito |
| D-11 | Backend de observabilidade (OTLP) | Qualquer compatível com OTLP; avaliar custo |
| D-12 | i18n / idioma padrão da UI | Preparar chaves de i18n desde o início; idioma a definir |
| D-13 | Catálogo inicial de mudanças que a IA pode propor | Sem operações destrutivas na v1 |
| D-14 | Privacidade do conteúdo das notificações (título no payload) | Setting "ocultar conteúdo" |
| D-15 | Requisitos legais de dados pessoais (ex.: LGPD) — exportação/exclusão | Projetar exclusão e exportação desde já; confirmar obrigações |
