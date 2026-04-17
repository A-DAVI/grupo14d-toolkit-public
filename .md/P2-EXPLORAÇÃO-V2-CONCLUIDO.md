# Exploração Telemetria — V2 (estado em 2026-04-17 após nova leva de commits)

> Este documento substitui a versão anterior do V2. Entre a última exploração e esta, praticamente todo o ecossistema de RPAs recebeu commits (2026-04-17, 09:30–11:03). Hoje há **uma única geração de lib cliente** em todos os RPAs, **uma única URL de servidor** (`<MONITOR_INTERNAL_URL>`) e **15 de 15 RPAs reportam telemetria** com `X-Api-Key`. O MONITOR-RPA também evoluiu (refactor `apps/api` + `apps/web`, novos endpoints de gestão e analytics). Fonte de verdade: `/Users/davicassoli/Trabalho/14D-PROJETOS/`.

## 0. Diff vs V2 anterior (o que mudou em 2026-04-17)

| Tópico | V2 anterior | V2 agora |
|--------|-------------|----------|
| RPAs que reportam | 10 de 16 | **15 de 15** (RPA-CAIXA-E-LUCROS saiu do inventário; todos os demais reportam) |
| Gerações da lib `telemetry.py` | **2** coexistindo (nova com auth, antiga sem auth) | **1 só** — "nova geração" em todos os clientes (X-Api-Key, `datetime.now(timezone.utc)`, `_resolve_api_key`, `config.ini`) |
| URL do servidor | 2 URLs (`192.168.1.3:8000` + <hosting>) | **1 URL** — `<MONITOR_INTERNAL_URL>` em 100% dos RPAs. <hosting> abandonado (TROTS-ALT-DE-ENTRADA foi revertido, commit `4713619`) |
| RPA-Cartorios | Telemetria instanciada mas não chamada (código morto) | **Ativo** — lib migrada + `config-global.ini` com `rpa_name=cartorios` e `server_url=<MONITOR_INTERNAL_URL>` (commit `2026-04-17 10:40`) |
| RPA-BALANCETES | Não reportava | **Passou a reportar** — `rpa_name=balancete`, lib nova (commit `2026-04-17 10:39`) |
| RPA-DROGARIA | Não reportava | **Passou a reportar** — `rpa_name=drogaria`, lib nova (commit `2026-04-17 10:34`) |
| RPA-SISTEMA-GRUPO14D | Conversor local, sem telemetria | **Passou a reportar** — `rpa_name=sistema-grupo14d`, lib nova (commit `2026-04-17 10:30`) |
| RPA-TROTS | Não reportava | **Passou a reportar** — `rpa_name=trots`, lib nova (commit `2026-04-17 10:59`) |
| RPA-TROTS-IMPORT | Não reportava | **Passou a reportar** — via `config/pipeline.yml`, `rpa_name=trots_import_nfse_tomados` (commit `2026-04-17 10:32`) |
| RPA-XML-HOSPITAL | Antiga geração | **Migrado** — `rpa_name=hospital-xml`, lib nova (commit `2026-04-17 10:27`) |
| RPA-SITTAX | Antiga geração (sem auth) | **Migrado** — `rpa_name=sittax`, lib nova + X-Api-Key (commit `2026-04-17 10:25`) |
| RPA-REBNIC | Antiga geração | **Migrado** — `rpa_name=rebnic`, lib nova (commit `2026-04-17 10:34`) |
| RPA-REBNIC-ACUMULADORES | Antiga geração | **Migrado** — `rpa_name=Rebnic-Acumuladores`, lib nova (commit `2026-04-17 11:02`) |
| Heartbeat IRPF | 60s artesanal em `fluxo_caixa_irpf.py` | **Novo evento `watcher_heartbeat`** em `src/watcher/pdf_watcher.py:118-140` a cada **300s** (5 min), com payload estruturado (`records`, `totalRecords`, `description`, `started_at`, `last_file`, `last_renamed_at`, `errors_since_start`). Também existe em `entregues_reconciler.py:52` e `sheets_sync_watcher.py:242`. Inclui `watcher_started` no boot. |
| MONITOR-RPA | Fastify+Node+Neon, monolito em `src/` | **Refactor `apps/api` + `apps/web`** (commit `5b69936`), novos endpoints de machines, responsibles, business analytics, queue overview/stream, e módulo `services/queueOps.ts` (log ring + SSE dedicado) |
| Tabelas novas no Neon | — | `rpa_machines`, `queue_entregues_publications`, `agent_sessions`, `rpa_cleared_months` |

---

## Resumo executivo

- **Servidor ativo**: `MONITOR-RPA` em `14D-PROJETOS/MONITOR-RPA/apps/api` — Fastify 5 + <postgres>, exige `X-Api-Key` em `POST /events`. Tabela `rpa_events` inalterada no contrato; periféricos (machines, responsibles, cleared months, business-analytics, agents, queue) expandidos.
- **15 RPAs mapeados, 15 reportando** com `telemetry.py` homogêneo (uma só geração).
- **Contrato v1 já está, na prática, implementado** no código — falta apenas: documentar, versionar (`contract_version`), consolidar a lib em pacote compartilhado, e padronizar heartbeat (hoje existe só no IRPF).
- **Pontos residuais de fragmentação**: (1) a lib é idêntica mas vive copiada em cada repo; (2) nomenclatura de `rpa` ainda mistura `kebab-case`, `snake_case` e `PascalCase`; (3) Cartorios ainda reporta como um só RPA (`rpa=cartorios`) com 6 empresas distintas; (4) `heartbeat` só no IRPF (3 watchers); (5) sem `contract_version` em nenhum payload.
- **URL única**: `<MONITOR_INTERNAL_URL>` em todos os clientes. <hosting> (`<monitor-prod>.up.railway.app`) não é mais usado por nenhum RPA (inclusive TROTS-ALT-DE-ENTRADA foi revertido).

---

## 1. MONITOR-RPA (servidor)

Código fonte novo em `/Users/davicassoli/Trabalho/14D-PROJETOS/MONITOR-RPA/apps/api/`. O front vive em `apps/web/`. Refactor em `5b69936`.

### 1.1 Endpoints (todos em `apps/api/server.ts` + `routes/*.ts`)

**Ingestão e estado principal** (`server.ts`)

| Método | Rota | Linha | Auth | Notas |
|--------|------|-------|------|-------|
| POST   | `/events` | 223 | `X-Api-Key` (env `<RPA_API_KEY_ENV>`) | Ingestão única de telemetria. Schema Fastify com `additionalProperties: true` |
| GET    | `/api/status` | 261 | pública | snapshot `Map<rpa, RpaStatus>` em memória |
| GET    | `/api/stream` | 272 | pública | SSE, heartbeat 30s, `rateLimit: false` |
| GET    | `/api/events?horas=N` | 314 | pública | histórico janelado |
| GET    | `/api/events/monthly?year=&month=` | 338 | pública | cache 60s (`monthlySummaryCache`) |
| GET    | `/api/machines` | 405 | pública | lista máquinas com metadados, aplicando `EXCLUDED_MACHINES` |
| PUT    | `/api/machines/:machine` | 410 | pública¹ | `{ excluded: bool, label? }` — novo |
| GET    | `/api/rpa-config` | 441 | pública | domínio + alias + responsável por RPA |
| PUT    | `/api/rpa-config/:rpa/domain` | 455 | pública¹ | |
| PUT    | `/api/rpa-config/:rpa/alias` | 470 | pública¹ | |
| PUT    | `/api/rpa-config/:rpa/responsible` | 486 | pública¹ | novo |
| GET    | `/api/rpa-config/responsibles` | 502 | pública | lista distintos |
| POST   | `/api/rpa-config/:rpa/clear` | 507 | pública¹ | marca mês/ano como limpo |
| DELETE | `/api/rpa-config/:rpa/clear` | 525 | pública¹ | |
| GET    | `/api/business-analytics?days=N` | 543 | pública | agregado por responsável, setor, timeline — novo |
| GET    | `/health` | 652 | pública | smoke test |

¹ Rotas administrativas sem gate hoje — observação da Seção 3.

**Fila (`apps/api/routes/queue.ts`)**

| Método | Rota | Linha | Auth | Notas |
|--------|------|-------|------|-------|
| POST   | `/queue/jobs`       | 236 | token no body (`<QUEUE_TOKEN_ENV>`) | Publica job "entregues"; valida dedup em `queue_entregues_publications` |
| GET    | `/queue/status`     | 331 | pública | legado |
| GET    | `/api/queue/overview` | 339 | pública | estado operacional para a UI |
| GET    | `/api/queue/stream` | 368 | pública | SSE de logs da fila/telemetria (via `queueOps`) |

**Agents (`apps/api/routes/agents.ts`)**

| Método | Rota | Linha | Auth | Notas |
|--------|------|-------|------|-------|
| POST   | `/api/agents`       | 24  | `X-Agent-Role: admin` | cria sessão |
| GET    | `/api/agents`       | 53  | pública | lista |
| GET    | `/api/agents/:id/stream` | 60 | pública | SSE por sessão |
| POST   | `/api/agents/:id/task`   | 100 | `X-Agent-Role: admin` | envia task |
| DELETE | `/api/agents/:id`        | 128 | `X-Agent-Role: admin` | encerra |

### 1.2 Schema de eventos (`POST /events`)

Definido em `server.ts:203-219`. Campos obrigatórios: `rpa`, `event`, `session_id`, `machine`, `empresa`, `timestamp`. Opcionais com tipo explícito: `status` (string), `duracao_segundos` (number 0–86400). `additionalProperties: true` → qualquer outro campo vai para a coluna JSONB `rpa_events.extra`.

### 1.3 Persistência (`apps/api/db.ts`)

Tabela `rpa_events` (`db.ts:77-91`) **inalterada** em relação ao P1 / V2 anterior:

```sql
id SERIAL PRIMARY KEY,
rpa TEXT, event TEXT, session_id TEXT, machine TEXT, empresa TEXT,
timestamp TIMESTAMPTZ, received_at TIMESTAMPTZ DEFAULT NOW(),
status TEXT, duracao_segundos NUMERIC,
extra JSONB DEFAULT '{}'::jsonb
```

`KNOWN_FIELDS` em `db.ts:157-160` separa payload top-level do `extra`. `insertEvent` em `db.ts:175-197`. Índices: `(timestamp DESC)` e `(rpa, timestamp DESC)`.

Tabelas periféricas criadas por `initDb()`:
- `rpa_config` (`db.ts:104-110`) — domain/active/alias/responsible.
- `rpa_machines` (criada em `createMachinesTable()`).
- `rpa_cleared_months` (`db.ts:122-130`).
- `queue_entregues_publications` (`db.ts:132-143`) — dedup persistente de jobs.
- `agent_sessions` (`createAgentSessionsTable()`).

### 1.4 Autenticação

Três mecanismos (todos sem mudança de contrato desde o V2 anterior):

1. **X-Api-Key** (`server.ts:190-198`) — obrigatório em `POST /events`. Env `<RPA_API_KEY_ENV>`.
2. **Token no body** (`routes/queue.ts:257`) — obrigatório em `POST /queue/jobs`. Env `<QUEUE_TOKEN_ENV>`.
3. **X-Agent-Role: admin** — obrigatório em endpoints de agentes que criam/mutam.

### 1.5 Variáveis de ambiente relevantes

Encontradas em `apps/api/*.ts`:

- `<RPA_API_KEY_ENV>` — chave única da ingestão (`server.ts:88`).
- `<QUEUE_TOKEN_ENV>` — autenticação de `POST /queue/jobs` (`routes/queue.ts:22`).
- `PORT` (default `8000`), `QUEUE_ENABLED` (default `true`), `FRONTEND_URL`, `NODE_ENV`, `TRUST_PROXY` — operacionais (`server.ts:55-56, 85-88, 134-136`).
- `EXCLUDED_MACHINES` — lista CSV de máquinas a ocultar do dashboard (`db.ts:214`), cache 30s (`db.ts:228`).
- `OBSIDIAN_API_URL`, `OBSIDIAN_API_KEY` — usados pelo `agentService.ts:58-59`.
- `APPS_SCRIPT_ENTREGUES_URL`, `APPS_SCRIPT_TOKEN` — usados pelo worker de entregues (`services/workers/entregues.worker.ts:29`, `routes/queue.ts:262`).

### 1.6 Módulo novo: `services/queueOps.ts`

Ring buffer de 500 logs em memória + SSE dedicado. `recordQueueLog()` recebe entradas de três fontes (`telemetry` | `queue` | `worker`) e alimenta `workerSnapshot` + `/api/queue/stream`. `recordQueueTelemetryEvent()` é chamado dentro do `POST /events` (`server.ts:244`), mapeando eventos de RPA para o fluxo operacional (filtra `RELEVANT_RPAS = { rpa-irpf-sync, rpa-irpf-watcher, rpa-irpf-entregues-reconciler }`).

### 1.7 SSE / heartbeat servidor→cliente

`server.ts:272-311` (`/api/stream`) envia `ping` a cada 30s, `status` no connect e a cada evento recebido. Separado do `/api/queue/stream` (fila).

---

## 2. RPAs (clientes)

### 2.1 Inventário

16 diretórios em `14D-PROJETOS/` — 1 é o servidor, 15 são RPAs. **Todos reportam**. Resumo (todos apontam para `<MONITOR_INTERNAL_URL>`, todos com `enabled=true` em `config.ini`, todos usam lib "nova geração" com `X-Api-Key`):

| # | Repo | `rpa_name` | Lib em | Heartbeat? | Observações |
|---|------|------------|--------|------------|-------------|
| 1  | RPA-BALANCETES | `balancete` | `src/telemetry.py` | não | começou a reportar hoje (commit 10:39) |
| 2  | RPA-COBRANCA-AUTOMATICA | `cobranca-automatica` | `telemetry.py` (raiz) | não | `config/config.example.ini` é o template de deploy |
| 3  | RPA-Cartorios | `cartorios` | `telemetry.py` (raiz) | não | **reativado**; `config-global.ini` substitui o config.ini ausente; 6 empresas, todas sob o mesmo `rpa` |
| 4  | RPA-Conferencia-NFs-e | `conferencia-nfse` | `telemetry.py` (raiz) | não | |
| 5  | RPA-DROGARIA | `drogaria` | `telemetry.py` (raiz) | não | commit anterior virou RPA ativo |
| 6  | RPA-IRPF | `rpa-irpf` (+ `rpa-irpf-watcher`, `rpa-irpf-entregues-reconciler`, `rpa-irpf-sync`) | `telemetry.py` (raiz) | **sim** — `watcher_heartbeat` a cada 300s em `pdf_watcher.py:118`, `entregues_reconciler.py:52`, `sheets_sync_watcher.py:242` | heartbeat antigo de 60s em `fluxo_caixa_irpf.py` foi substituído pelo watcher |
| 7  | RPA-LANCAMENTOS-REBNIC | `lancamentos-rebnic` | `src/adapters/telemetry.py` | não | única lib em `adapters/` (resto dos repos põe na raiz ou `src/`) |
| 8  | RPA-REBNIC | `rebnic` | `telemetry.py` (raiz) | não | migrado hoje |
| 9  | RPA-REBNIC-ALTERACAO-ACUMULADOR-DOMINIO | `Rebnic-Acumuladores` | `src/telemetry.py` | não | **único com PascalCase/hífen** no `rpa_name`; emite eventos extras `automation_started/finished` |
| 10 | RPA-SISTEMA-GRUPO14D | `sistema-grupo14d` | `telemetry.py` (raiz) | não | passou a reportar hoje |
| 11 | RPA-SITTAX | `sittax` | `src/telemetry.py` | não | migrado hoje (commit 10:25) |
| 12 | RPA-TROTS | `trots` | `telemetry.py` (raiz) | não | passou a reportar hoje |
| 13 | RPA-TROTS-ALT-DE-ENTRADA | `trots_alt_entrada` | `src/utils/telemetry.py` | não | **revertido de <hosting> para IP local** (commit 4713619); usa snake_case |
| 14 | RPA-TROTS-IMPORT | `trots_import_nfse_tomados` | `src/telemetry.py` | não | configuração via `config/pipeline.yml` (bloco `telemetry:`) em vez de `config.ini` |
| 15 | RPA-XML-HOSPITAL | `hospital-xml` | `src/telemetry.py` | não | migrado hoje |

### 2.2 Padrão da lib "nova geração" (único em uso)

Mesma classe `Telemetry` em todos os 15 repos, com variações mínimas de path. Características:

- **Datas UTC**: `datetime.now(timezone.utc)` (sem naive datetime).
- **Autenticação**: header `X-Api-Key`. Resolução em cascata por `_resolve_api_key()`:
  1. argumento explícito;
  2. env vars `<RPA_API_KEY_ENV>`, `MONITOR_<RPA_API_KEY_ENV>`, `<RPA_MONITOR_API_KEY_ENV>`;
  3. arquivo (`<api-key-file>`, `api_key.txt`, `<secrets-api-key-file>`).
- **Config via `config.ini`** (exceto TROTS-IMPORT, que usa `pipeline.yml`; e Cartorios, que usa `config-global.ini`): bloco `[telemetry]` com `enabled`, `server_url`, `rpa_name`.
- **Timeout**: 5s (era 3s na geração antiga).
- **Envio**: thread `daemon=True` + fallback silencioso em falha. Sem retry exponencial.
- **`session_id`**: UUID completo (`uuid.uuid4()`), não mais truncado em 8 chars.
- **Eventos emitidos**: `rpa_started`, `rpa_finished`, `rpa_error`. REBNIC-ACUMULADORES adiciona `automation_started`, `automation_finished`. IRPF adiciona `watcher_started`, `watcher_heartbeat`.

### 2.3 Detalhe do IRPF (único com heartbeat)

Em `RPA-IRPF/src/watcher/pdf_watcher.py:30` a constante `INTERVALO_HEARTBEAT = 300` define o intervalo. O loop `_heartbeat_loop` (linha 118) chama `telemetry._send("watcher_heartbeat", empresa="IRPF", extra={...})` com payload estruturado:

```
records, totalRecords, description, started_at, last_file, last_renamed_at, errors_since_start
```

Dois outros watchers (`entregues_reconciler.py:52`, `sheets_sync_watcher.py:242`) também enviam `watcher_heartbeat` — é o único evento de heartbeat real no ecossistema.

---

## 3. Divergências e inconsistências

1. **Nomenclatura do `rpa` ainda heterogênea**: `kebab-case` (maioria), `snake_case` (`trots_alt_entrada`, `trots_import_nfse_tomados`), `PascalCase-Hybrid` (`Rebnic-Acumuladores`). Quebra ordenação e dificulta padrão.
2. **Cartorios é um `rpa` só para 6 empresas**: continua como no P1. O servidor não distingue unidades no eixo RPA — só no campo `empresa`.
3. **REBNIC-ACUMULADORES emite eventos fora do enum de fato** (`automation_started`, `automation_finished`). Servidor aceita (não há enum), mas não é documentado.
4. **Lib `telemetry.py` duplicada em 15 repos** — embora idêntica na estrutura, é cópia. Drift futuro é questão de tempo. Nenhum pacote compartilhado existe.
5. **Heartbeat só no IRPF**. Demais RPAs ficam invisíveis para o watcher do dashboard entre `rpa_started` e `rpa_finished`. Se travam sem `rpa_error`, não há sinal.
6. **Sem `contract_version` no payload**. Migrações futuras ficam sem alavanca.
7. **Endpoints admin (`PUT /api/machines/*`, `PUT /api/rpa-config/*`, `POST/DELETE /api/rpa-config/:rpa/clear`) estão abertos** — nenhum gate de auth. `GET /api/stream`, `/api/events`, `/api/business-analytics` também são públicos.
8. **Sem migrations versionadas** no Neon. Schema inline em `db.ts:77-151`. Alterações são `ALTER TABLE … ADD COLUMN IF NOT EXISTS` sem histórico.
9. **Sem validação de enum de `event` ou `status`** no `eventSchema`. Qualquer string cabe.
10. **Sem política de retenção** para `rpa_events`. Cresce sem limite; relevante em Neon serverless.
11. **`config.ini` vs `config-global.ini` vs `pipeline.yml`** — três formatos coexistindo. RPA-TROTS-IMPORT (YAML) e RPA-Cartorios (config-global.ini) quebram o padrão.
12. **Nenhuma variável de ambiente canônica para `server_url`** — hoje o valor está hardcoded em `config.ini`/`pipeline.yml`. Mudança de IP/URL exige edit em 15 arquivos.

---

## 4. Trechos de código relevantes

Servidor:

- `MONITOR-RPA/apps/api/server.ts:190-198` — `requireApiKey` (bloqueia `POST /events` sem `X-Api-Key`).
- `MONITOR-RPA/apps/api/server.ts:203-219` — `eventSchema` (Fastify JSON schema, `additionalProperties: true`).
- `MONITOR-RPA/apps/api/server.ts:223-258` — handler de `POST /events`, incluindo `recordQueueTelemetryEvent(payload)` (integração queueOps).
- `MONITOR-RPA/apps/api/server.ts:272-311` — SSE `/api/stream` com heartbeat 30s.
- `MONITOR-RPA/apps/api/server.ts:405-438` — endpoints de machines (GET + PUT).
- `MONITOR-RPA/apps/api/server.ts:441-540` — bloco de `rpa-config` (domain/alias/responsible/clear).
- `MONITOR-RPA/apps/api/server.ts:543-650` — `business-analytics` (agregados por responsável/setor/timeline).
- `MONITOR-RPA/apps/api/db.ts:77-91` — DDL `rpa_events`.
- `MONITOR-RPA/apps/api/db.ts:132-151` — DDL `queue_entregues_publications`.
- `MONITOR-RPA/apps/api/db.ts:157-160` — `KNOWN_FIELDS`.
- `MONITOR-RPA/apps/api/db.ts:175-197` — `insertEvent`.
- `MONITOR-RPA/apps/api/routes/queue.ts:236-330` — `POST /queue/jobs` com dedup via `queue_entregues_publications`.
- `MONITOR-RPA/apps/api/routes/queue.ts:339-380` — `/api/queue/overview` e `/api/queue/stream`.
- `MONITOR-RPA/apps/api/services/queueOps.ts:96-118` — `recordQueueLog`.
- `MONITOR-RPA/apps/api/services/workers/entregues.worker.ts:29` — leitura de `APPS_SCRIPT_ENTREGUES_URL`.

Clientes (padrão — válido para todos, exemplificando com alguns):

- `RPA-Cartorios/telemetry.py` — classe `Telemetry` da nova geração.
- `RPA-IRPF/src/watcher/pdf_watcher.py:30` — `INTERVALO_HEARTBEAT = 300`.
- `RPA-IRPF/src/watcher/pdf_watcher.py:118-140` — loop de heartbeat.
- `RPA-IRPF/src/watcher/pdf_watcher.py:151-162` — `watcher_started` no boot.
- `RPA-REBNIC-ALTERACAO-ACUMULADOR-DOMINIO/src/telemetry.py` — exemplo com eventos `automation_*`.
- `RPA-TROTS-IMPORT/config/pipeline.yml` — `telemetry:` via YAML.
- `RPA-Cartorios/config-global.ini` — exemplo de `rpa_name=cartorios` fora do `config.ini` padrão.

---

## 5. Contexto do vault

(Sem mudanças estruturais relevantes desde o V2 anterior — o vault continua como referência histórica, não está sincronizado com o repo.)

- `grupo14d-obsidian-vault/Contextos/CONTEXTO - MONITOR-RPA.md` — stack e visão geral.
- `grupo14d-obsidian-vault/Projetos/MONITOR-RPA/Home.md` — menciona endpoint futuro `POST /api/telemetria/org-agent` (escopo ORG-AGENT, separado do contrato dos RPAs).
- `grupo14d-obsidian-vault/Projetos/MONITOR-RPA/Modulos/Pronto/02 - Stream de Eventos e Backend SSE.md` — decisões de heartbeat 30s e rate limit 300 req/min.
- `grupo14d-obsidian-vault/Projetos/RPA - COBRANÇA AUTOMATICA/Modulos/Pronto/08 - Telemetria.md` — único módulo "telemetria" explicitamente documentado; referência natural para o template v1 de onboarding.
- ADRs `0001` (Fastify), `0002` (Neon), `0003` (<process-manager>) existem apenas em `MONITOR-RPA/` — **não estão espelhadas no vault**.

---

## 6. Recomendação de contrato v1

Hoje a defasagem entre "o que o código faz" e "o que o contrato v1 deveria exigir" ficou pequena. A proposta abaixo **formaliza o que já está rodando** e fecha os buracos residuais:

### 6.1 Payload obrigatório

```json
{
  "contract_version": "v1",
  "rpa": "<slug kebab-case>",
  "event": "rpa_started | rpa_finished | rpa_error | heartbeat | automation_started | automation_finished | watcher_started | watcher_heartbeat",
  "session_id": "<uuid4>",
  "machine": "<hostname>",
  "empresa": "<nome ou slug>",
  "timestamp": "<ISO 8601 UTC, com timezone>"
}
```

### 6.2 Opcionais com tipo explícito

- `status` ∈ { `success`, `error`, `warning`, `running` }
- `duracao_segundos` ∈ `[0, 86400]`
- `detalhes` — objeto livre com convenções: `records`, `totalRecords`, `description`, `divergencias?`, `last_file?`, `errors_since_start?`

### 6.3 Autenticação

Manter `X-Api-Key` em `POST /events`. **Adicionar**: chave por RPA (hoje é única global em `<RPA_API_KEY_ENV>`). Permite revogar um RPA sem derrubar o ecossistema.

### 6.4 Heartbeat

Tornar obrigatório emitir `heartbeat` a cada 300s para RPAs de execução contínua (watchers, filas). RPAs de execução curta (< 5 min) ficam dispensados. Modelo canônico: o já adotado no IRPF.

### 6.5 Lib compartilhada

Extrair `telemetry.py` para pacote interno (`pip install -e` local ou repo próprio). Eliminar as 15 cópias. Versão do pacote → `contract_version` no payload.

### 6.6 Convenções de naming

- `rpa`: kebab-case obrigatório. Migrar os 3 snake_case (`trots_alt_entrada`, `trots_import_nfse_tomados`) e o PascalCase (`Rebnic-Acumuladores`).
- `empresa`: livre para já, mas documentar que a dashboard agrega por ela.

### 6.7 Servidor

Adicionar validação de `event` e `status` por enum no `eventSchema` (`server.ts:203-219`). Assim contrato v1 deixa de aceitar qualquer string.

---

## 7. Riscos e armadilhas

1. **Migrar `rpa_name` existentes quebra continuidade histórica**: `rpa_events.rpa` vira chave partida se renomearmos `trots_alt_entrada` → `trots-alt-entrada`. Mitigação: manter alias no `rpa_config.alias`, ou fazer backfill `UPDATE rpa_events SET rpa = ...`.
2. **Chave por RPA requer revisão do `requireApiKey`**: hoje compara contra env única. Mudança exige `HashSet<string>` ou lookup por tabela. Deploy coordenado para evitar 401 massivo.
3. **Lib em pacote compartilhado aumenta superfície de deploy**: hoje cada RPA é autossuficiente. Virar pacote cria dependência — versão travada no `requirements.txt` de cada RPA, ou `pip install -e ../grupo14d-telemetry` no build server.
4. **Endpoints admin abertos**: qualquer um na rede `192.168.1.3` pode `PUT /api/machines/:machine` ou `POST /api/rpa-config/:rpa/clear`. Se o servidor sair da rede interna (ex.: expor novamente via <hosting> ou VPN), vira vetor.
5. **`rpa_events` cresce sem retenção**: com 15 RPAs reportando + heartbeat IRPF a cada 5 min, volume vai subir rápido. Neon cobra por storage. Precisa política (ex.: partição mensal + `DELETE WHERE timestamp < NOW() - INTERVAL '90 days'`).
6. **Cartorios como RPA único**: se quisermos visão por cartório no dashboard, a única alavanca hoje é `empresa`. Passar para `rpa=cartorios-<apelido>` quebra a identidade; manter como está agrega bem, mas esconde granularidade.
7. **Heartbeat IRPF 300s**: se o watcher travar logo após o último heartbeat, o dashboard só detecta depois de 5 min. Intervalo menor pesa no banco; maior cria zona cega. Decisão de produto.
8. **TROTS-IMPORT com YAML** vs demais com INI: ao extrair a lib compartilhada, cobrir ambos ou padronizar em um só.
9. **Thread daemon + timeout 5s**: um RPA pode ter eventos silenciosamente perdidos se a rede oscilar e o processo terminar logo depois. Retry-fila local (SQLite/jsonl) elimina mas aumenta complexidade.
10. **`contract_version` não existe ainda em nenhum payload**: quando for adicionado, servidor precisa aceitar ausência por backcompat (e registrar `v0` implícito).

---

## 8. Perguntas abertas para o Davi

1. **Pacote compartilhado de `telemetry.py`**: preferência por (a) repo próprio `grupo14d-telemetry` com pip install editable, (b) sub-diretório em `grupo14d-toolkit/` vendored em cada RPA, ou (c) continuar com cópias e um script de sincronização?
2. **Nomenclatura de `rpa`**: pode migrar `trots_alt_entrada`, `trots_import_nfse_tomados` e `Rebnic-Acumuladores` para kebab-case? Se sim, faço backfill em `rpa_events` ou mantenho alias?
3. **Chave por RPA**: faz sentido a API key passar a ser granular (uma por RPA), ou a chave única está boa enquanto o servidor estiver só na rede interna?
4. **Heartbeat obrigatório**: aceitável padronizar em 300s para watchers longos, e ficar opcional para RPAs curtos? Ou prefere intervalo menor (60s)?
5. **`contract_version` no payload**: ok adicionar como campo obrigatório em v1, com servidor aceitando payloads sem o campo como `v0`?
6. **Cartorios**: manter `rpa=cartorios` e diferenciar por `empresa`, ou quebrar em `rpa=cartorios-<apelido>`? Impacta visão histórica.
7. **Enum de `event`/`status`**: ok travar o schema do servidor para recusar strings fora do enum v1, ou preferir aceitar e só alertar?
8. **Retenção em `rpa_events`**: política para seguir (janela de 90 dias? 180? particionamento?)?
9. **Endpoints admin sem auth**: posso colocar `X-Agent-Role: admin` (ou similar) em `PUT /api/machines/*`, `PUT /api/rpa-config/*`, `POST/DELETE /api/rpa-config/:rpa/clear`, ou isso quebra alguma ferramenta atual?
10. **MONITOR-RPA no <hosting>**: se nenhum RPA aponta mais para lá, dá para desligar o deploy <hosting> ou ele ainda serve ao frontend público/externo?
