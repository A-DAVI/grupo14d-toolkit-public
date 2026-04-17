# Exploração Telemetria — Estado Atual

## Resumo executivo

- **Servidor ativo**: `MONITOR-RPA` (Fastify + <postgres>, Node/TS), deployado em `<MONITOR_PROD_URL>`. Última atividade: 2026-04-15. Valida payload com JSON Schema do Fastify, persiste em tabela `rpa_events`, exige header `X-Api-Key` em `POST /events`.
- **Servidor legado**: `Server-Monitoracao` (FastAPI + JSONL), 1 único commit em 2026-02-19, sem auth, sem validação. Tratar como referência histórica.
- **Cobertura real**: apenas **2 de 6** RPAs reportam telemetria hoje — `RPA-Cartorios` (6 cartórios compartilhando um único identificador) e `RPA-REBNIC-Acumuladores`. Ambos usam cópias quase idênticas de `telemetry.py`, com drift já começando.
- **Principal dor**: clientes **não enviam** `X-Api-Key`, mas o servidor documenta exigência. Ou a env `<RPA_API_KEY_ENV>` está vazia em prod (caindo no caminho feliz), ou existe um proxy/versão diferente. Esta divergência trava qualquer decisão de contrato v1.
- **Recomendação preliminar**: antes de desenhar v1, confirmar estado real de auth em prod; consolidar lib em pacote compartilhado; decidir identificação por-cartório no Cartorios; adicionar heartbeat.

---

## 1. MONITOR-RPA (servidor)

Stack: **Fastify 5 + Node.js/TypeScript**, monorepo `apps/api` + `apps/web`, Postgres serverless via **<postgres-driver>**, frontend React 18 + Vite 6. Deploy via Docker + <process-manager>, hospedado em <hosting>.

Estrutura relevante:

```
MONITOR-RPA/
├── apps/api/
│   ├── server.ts             orquestrador (endpoints + SSE)
│   ├── db.ts                 schema + operações Postgres
│   ├── helpers.ts            tipos TelemetryEvent, RPAStatus
│   ├── middleware/agentAuth.ts
│   ├── routes/
│   │   ├── agents.ts         módulo AGENTS (spawn/task/SSE)
│   │   └── queue.ts          fila pg-boss (entregues)
│   └── services/
├── apps/web/                 dashboard React
├── TELEMETRY_DATA_CONTRACT.md contrato documentado (2026-03-02)
└── CONTEXT.md
```

### 1.1 Endpoints

Total catalogado: **25 endpoints**. Os relevantes pra telemetria:

| Método | Rota | Arquivo:linha | Auth | Observação |
|--------|------|---------------|------|------------|
| POST | `/events` | `server.ts:223` | `X-Api-Key` | Ingestão principal de eventos de RPA |
| GET | `/api/status` | `server.ts:261` | — | Snapshot do estado atual dos RPAs |
| GET | `/api/stream` | `server.ts:272` | — | SSE, heartbeat servidor→client a cada 30s |
| GET | `/api/events?horas=N` | `server.ts:314` | — | Histórico (1-168h, default 24h) |
| GET | `/api/monthly-summary?year=Y&month=M` | `server.ts:338` | — | Resumo mensal agregado |
| GET | `/api/machines` | `server.ts:405` | — | Máquinas registradas |
| PUT | `/api/machines/:machine` | `server.ts:410` | — | Admin de exclusão/label |
| GET/PUT | `/api/rpa-config/*` | `server.ts:441-540` | — | Domínio, alias, responsável |
| GET | `/api/business-analytics?days=N` | `server.ts:543` | — | KPIs, timeline |
| POST | `/queue/jobs` | `routes/queue.ts:235` | token body | Fila pg-boss (IRPF sync) |
| GET | `/api/queue/stream` | `routes/queue.ts:355` | — | SSE de logs de fila |
| POST/GET/DELETE | `/api/agents*` | `routes/agents.ts` | `X-Agent-Role: admin` | Escritório Claude Code |
| GET | `/health` | `server.ts:652` | — | Healthcheck |

### 1.2 Schema de eventos

**Validação Fastify** (`server.ts:204-258`):

- **Obrigatórios**: `rpa`, `event`, `session_id`, `machine`, `empresa`, `timestamp`
- **Opcionais validados**: `status` (string ≤50), `duracao_segundos` (0-86400)
- **`additionalProperties: true`**: qualquer campo extra é aceito e vai pra coluna JSONB `extra`
- **Limites**: `rpa`, `event` ≤100 chars; `session_id`, `machine`, `empresa` ≤200 chars

**Schema SQL da tabela** (`db.ts:78-91`):

```sql
CREATE TABLE rpa_events (
  id               SERIAL PRIMARY KEY,
  rpa              TEXT        NOT NULL,
  event            TEXT        NOT NULL,
  session_id       TEXT        NOT NULL,
  machine          TEXT        NOT NULL,
  empresa          TEXT        NOT NULL,
  timestamp        TIMESTAMPTZ NOT NULL,
  received_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  status           TEXT,
  duracao_segundos NUMERIC,
  extra            JSONB       NOT NULL DEFAULT '{}'::jsonb
);
```

Outras tabelas: `rpa_config` (PK `rpa`), `rpa_machines` (PK `machine`), `rpa_cleared_months` (PK `rpa, year, month`), `queue_entregues_publications`, `pgboss.job`, `agent_sessions`.

**Sem migrations versionadas**: schema é criado inline com `CREATE TABLE IF NOT EXISTS` + `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`. Histórico opaco.

### 1.3 Autenticação

Três mecanismos distintos:

- **`POST /events`**: header `X-Api-Key` comparado com `process.env.<RPA_API_KEY_ENV>` (`server.ts:190-198`). Chave global, compartilhada entre todos os RPAs.
- **`POST /queue/jobs`**: token no body, comparado com `process.env.<QUEUE_TOKEN_ENV>` (`routes/queue.ts:257`). Separado, só pra fila IRPF.
- **Admin de agents**: header `X-Agent-Role: admin` (`middleware/agentAuth.ts:9-20`). Sem JWT/session — forjável.

Todos os GETs (`/api/status`, `/api/events`, `/api/machines`, config, analytics) são **públicos**, dependem de firewall de rede.

### 1.4 Observações

- Enum de `status` **não é validado** — aceita qualquer string; frontend espera apenas `success`, `error`, `warning` (`helpers.ts:46-49`).
- `rpa_events` **sem política de retenção** — tabela cresce indefinidamente.
- Estado em memória (`Map<string, RPAStatus>`) reconstruído do DB no boot (`server.ts:684-690`), SSE clients perdidos no reboot.
- Cache com TTL curto (30s config, 60s resumo mensal) + invalidação manual nos upserts.
- Sem logging estruturado (apenas `console.log` com timestamp ISO).
- Rate limit: 300 req/min global, SSE isento.

---

## 2. RPAs (clientes)

### 2.1 Inventário

| RPA | Caminho | Reporta? | Heartbeat? | Lib usada |
|-----|---------|----------|------------|-----------|
| SITTAX | `RPA---SITTAX/` | Não | — | — |
| Cartorios (6 cartórios) | `RPA-Cartorios/` | **Sim** | Não | `telemetry.py` próprio (cópia local) |
| FLUXO-CAIXA | `RPA-FLUXO-CAIXA/` | Não | — | — (Streamlit dashboard, não é RPA de automação) |
| REBNIC-Acumuladores | `RPA-REBNIC-ALTERACAO-ACUMULADOR-DOMINIO/` | **Sim** | Não | `telemetry.py` próprio (cópia local, quase idêntica à do Cartorios) |
| Capt-V1 | `Sistema-Capt-Empresarial/CaptV1/` | Não | — | — |
| Capt-V2 | `Sistema-Capt-Empresarial/v2/` | Não | — | — (FastAPI+React, sem integração) |

### 2.2 Detalhamento por RPA

#### RPA-Cartorios

- **Stack**: Python 3 + Tkinter + ttkbootstrap + pandas, empacotado com PyInstaller (`.spec`).
- **Subprojetos**: `cartorio_aline`, `cartorio_dermeval`, `cartorio_kalore`, `cartorio_marumbi`, `cartorio_mayra`, `cartorio_tiago` — **todos compartilham a mesma `telemetry.py`** no raiz.
- **URL alvo**: `<MONITOR_PROD_URL>` (**hardcoded** em `ui/main_window.py:21-27`, sem env var).
- **Identificação**: `rpa_name="cartorios"` — **genérico para todos os 6**; o servidor não distingue um cartório do outro via campo `rpa`, só via `empresa`.
- **`session_id`**: `uuid.uuid4()[:8]`, novo a cada execução.
- **`machine`**: `socket.gethostname()`.
- **Eventos emitidos**: `rpa_started`, `rpa_finished` (com `status: success|error`), `rpa_error`.
- **Auth**: **não envia** `X-Api-Key`. POST só com `Content-Type: application/json`.
- **Assíncrono**: cada POST roda em `threading.Thread(daemon=True)` com timeout 3s; falhas só geram `logger.debug`, não interrompem o RPA.
- **Heartbeat**: **não existe**.

#### RPA-REBNIC-Acumuladores

- **Stack**: Python 3 + Tkinter + pandas + pyautogui, PyInstaller.
- **URL alvo**: mesma do Cartorios, também hardcoded (`src/ui/main_window.py:8-14`).
- **Lib**: `src/telemetry.py` — **cópia quase idêntica** da do Cartorios, difere apenas em docstrings. Já há drift.
- **Identificação**: `rpa_name="Rebnic-Acumuladores"` — específica, diferente do padrão Cartorios.
- **Eventos extras**: além de `rpa_started/finished/error`, usa `automation_started` e `automation_finished` (`src/ui/main_window.py:293-299`). Esses eventos **não estão documentados** em `TELEMETRY_DATA_CONTRACT.md`.
- **`detalhes` suportado**: exemplos do contrato mostram `records`, `totalRecords`, `description`, `divergencias`.
- **Auth**: **não envia** `X-Api-Key`, igual Cartorios.
- **Heartbeat**: **não existe**.

---

## 3. Divergências e inconsistências

1. **Auth quebrada (ou não ativa em prod)**: servidor exige `X-Api-Key` (`server.ts:190-198`), mas nenhum dos 2 clientes ativos envia o header. Ou `<RPA_API_KEY_ENV>` está vazia em <hosting> (nesse caso `if (!key || key !== API_KEY)` ainda rejeita — precisa verificar código real), ou há um proxy à frente, ou está genuinamente 401 e ninguém percebeu porque o POST falha em thread daemon silenciosa.
2. **Nome do campo `rpa` vs `rpa_name`**: `TELEMETRY_DATA_CONTRACT.md` documenta `rpa_name`, código e clientes usam `rpa`. Contrato doc desatualizado.
3. **Status sem enum validado**: servidor aceita qualquer string, frontend assume apenas `success`/`error`/`warning`.
4. **Cartorios com identidade ambígua**: os 6 cartórios mandam `rpa="cartorios"`; a distinção entre eles só existe via `empresa`. Visão por-RPA no dashboard junta todos.
5. **REBNIC usa eventos `automation_*` não documentados**: `automation_started` e `automation_finished` não estão no contrato oficial.
6. **Lib `telemetry.py` duplicada**: Cartorios e REBNIC têm cópias independentes, já com drift em docstrings. Qualquer mudança futura precisa PR em N repos.
7. **Sem heartbeat no cliente**: dashboard depende de eventos ativos; daemons longos sem eventos parecem "mortos".
8. **`event` sem enum validado**: servidor aceita qualquer string; erros de digitação passam direto.
9. **URL do servidor hardcoded** em ambos os clientes — sem env var, sem `config.ini`. Troca de URL exige rebuild do `.spec`.
10. **Sem migrations versionadas** no MONITOR-RPA — schema evolutivo opaco.
11. **Sem política de retenção** em `rpa_events` — tabela cresce indefinidamente em Neon serverless.
12. **`detalhes` ≠ `extra`**: contrato fala em `detalhes` estruturado, mas servidor persiste tudo que não é KNOWN_FIELDS em `extra` JSONB. Reconciliação acontece em `helpers.ts:65` (`buildRpaStatus`) — não documentada.
13. **GETs públicos**: todos os endpoints de leitura (`/api/status`, `/api/events`, config, analytics) são acessíveis sem auth. Vazamento se rede comprometida.

---

## 4. Trechos de código relevantes

### Servidor — `requireApiKey` (`MONITOR-RPA/apps/api/server.ts:190-198`)

```typescript
function requireApiKey(request: FastifyRequest, reply: FastifyReply, done: () => void): void {
  const key = request.headers['x-api-key'];
  if (!key || key !== API_KEY) {
    log(`401 Unauthorized  POST /events  ip=${getClientIp(request)}`);
    reply.status(401).send({ ok: false, error: 'Unauthorized' });
    return;
  }
  done();
}
```

### Servidor — schema Fastify do `POST /events` (`server.ts:204-222`)

```typescript
const eventSchema = {
  body: {
    type: 'object',
    required: ['rpa', 'event', 'session_id', 'machine', 'empresa', 'timestamp'],
    properties: {
      rpa:              { type: 'string', maxLength: 100 },
      event:            { type: 'string', maxLength: 100 },
      session_id:       { type: 'string', maxLength: 200 },
      machine:          { type: 'string', maxLength: 200 },
      empresa:          { type: 'string', maxLength: 200 },
      timestamp:        { type: 'string', maxLength: 50  },
      status:           { type: 'string', maxLength: 50  },
      duracao_segundos: { type: 'number', minimum: 0, maximum: 86400 },
    },
    additionalProperties: true,
  },
};
```

### Servidor — `insertEvent` (`MONITOR-RPA/apps/api/db.ts:175-197`)

```typescript
export async function insertEvent(payload: Record<string, any>): Promise<void> {
  const extra: Record<string, any> = {};
  for (const [key, val] of Object.entries(payload)) {
    if (!KNOWN_FIELDS.has(key)) extra[key] = val;
  }
  await sql`
    INSERT INTO rpa_events
      (rpa, event, session_id, machine, empresa, timestamp, received_at, status, duracao_segundos, extra)
    VALUES (
      ${payload.rpa}, ${payload.event}, ${payload.session_id},
      ${payload.machine}, ${payload.empresa}, ${payload.timestamp},
      NOW(), ${payload.status ?? null}, ${payload.duracao_segundos ?? null},
      ${JSON.stringify(extra)}
    )`;
}
```

### Servidor — SSE com heartbeat 30s (`server.ts:272-311`)

```typescript
server.get('/api/stream', { config: { rateLimit: false } }, async (request, reply) => {
  const res = reply.raw;
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.flushHeaders();
  // ... snapshot inicial ...
  sseClients.add(res);
  const heartbeat = setInterval(() => {
    try { res.write(': heartbeat\n\n'); } catch { /* cleanup */ }
  }, 30_000);
  request.raw.on('close', () => {
    clearInterval(heartbeat);
    sseClients.delete(res);
  });
});
```

### Cliente — `telemetry.py` `_send` (`RPA-Cartorios/telemetry.py:81-110`)

```python
def _send(self, event_type: str, empresa: str = "", extra: dict = None):
    if not self.enabled:
        return
    payload = {
        "rpa": self.rpa_name,
        "event": event_type,
        "session_id": self.session_id,
        "machine": self.machine,
        "empresa": empresa,
        "timestamp": datetime.now().isoformat(),
        **(extra or {}),
    }
    thread = threading.Thread(target=self._post, args=(payload,), daemon=True)
    thread.start()
```

### Cliente — `_post` sem X-Api-Key (`RPA-Cartorios/telemetry.py:99-125`)

```python
def _post(self, payload: dict):
    try:
        data = json.dumps(payload).encode("utf-8")
        req = Request(
            f"{self.server_url}/events",
            data=data,
            headers={"Content-Type": "application/json"},  # <-- sem X-Api-Key
            method="POST",
        )
        with urlopen(req, timeout=3):
            pass
    except (URLError, OSError) as e:
        logger.debug(f"[Telemetria] Falha ao enviar evento '{payload.get('event')}': {e}")
```

### Cliente — inicialização Cartorios (`RPA-Cartorios/ui/main_window.py:21-27`)

```python
from telemetry import Telemetry

TELEMETRY = Telemetry(
    rpa_name="cartorios",
    server_url="<MONITOR_PROD_URL>",
    enabled=True,
)
```

### Cliente — uso em Cartorios (`RPA-Cartorios/ui/main_window.py:199-219`)

```python
TELEMETRY.start(empresa=selection)
thread = threading.Thread(target=self._run_thread, args=(selection, mes), daemon=True)
thread.start()
# ... no final ...
TELEMETRY.finish(status="success", empresa=selection)
# ou:
TELEMETRY.finish(status="error", empresa=selection)
TELEMETRY.error(str(exc), empresa=selection)
```

### Cliente — eventos `automation_*` REBNIC (`RPA-REBNIC-.../src/ui/main_window.py:293-299`)

```python
TELEMETRY.automation_start(empresa=empresa)
try:
    # processamento com pyautogui
    TELEMETRY.automation_finish("success", empresa=empresa)
except Exception as e:
    TELEMETRY.automation_finish("error", empresa=empresa, detalhes={"error": str(e)})
```

### Cliente — `finish` com duração (`RPA-Cartorios/telemetry.py:43-59`)

```python
def finish(self, status: str = "success", empresa: str = "", detalhes: dict = None):
    duracao = None
    if self._start_time:
        duracao = round((datetime.now() - self._start_time).total_seconds(), 1)
    self._send(
        "rpa_finished",
        empresa=empresa,
        extra={**(detalhes or {}), "duracao_segundos": duracao, "status": status},
    )
```

---

## 5. Contexto do vault

Notas encontradas em `/Users/davicassoli/Trabalho/grupo14d-obsidian-vault/`:

- **`Contextos/CONTEXTO - MONITOR-RPA.md`** — stack, módulos, escritório de agentes integrado. Cita: *"Dashboard full-stack de monitoramento em tempo real dos processos RPA do Grupo 14D."*
- **`Projetos/MONITOR-RPA/Home.md`** — funcionalidades e integrações externas. Menciona endpoint pendente `POST /api/telemetria/org-agent` (escopo ORG-AGENT Electron, com `tokens_input/output` e `custo_estimado`) — **fora do escopo dos RPAs**.
- **`Projetos/MONITOR-RPA/Modulos/Pronto/02 - Stream de Eventos e Backend SSE.md`** — decisões registradas: heartbeat SSE 30s, rate limit 300 req/min, estado em memória reconstruído do DB no startup.
- **`Projetos/MONITOR-RPA/Modulos/Pronto/01 - Dashboard e Cartoes de Status.md`** — semântica batch vs daemon, watcher com heartbeat a cada 5 min. Registra *bug aberto*: timer de uptime reseta a cada heartbeat.
- **`Projetos/RPA - COBRANÇA AUTOMATICA/Modulos/Pronto/08 - Telemetria.md`** — **único exemplo prático documentado** de integração via classe `Telemetry`. Lifecycle: `start()` → `automation_start()` → `automation_finish()` → `error()` / `finish()`. Config em `config.ini` + env `<RPA_API_KEY_ENV>`. Status final: `success`/`warning`/`error`.
- **`Projetos/RPA - COBRANÇA AUTOMATICA/Modulos/Em Analise/02 - Telemetria e Confiabilidade.md`** — análise futura de auditabilidade, logs estruturados, screenshots.
- **`Projetos/Server-Monitoracao/Home.md`** — projeto legado, persiste em JSONL, dashboard via polling 5s.
- **`Mapas/Projetos.md`** — categoria *Infraestrutura e Monitoração* lista MONITOR-RPA, Server-Monitoracao, RPA-RELATORIOS, RPA-SISTEMA.
- **`Projetos/MONITOR-RPA/Mapas/Backlog.md`** — prioridades: alta (badge WARNING para watcher stale), média (retenção `pgboss.job` e logs), baixa (alertas proativos).

**ADRs** (0001 Fastify, 0002 <postgres>, 0003 <process-manager>) existem apenas no repositório MONITOR-RPA (em `docs/`), **não estão espelhados como notas no vault**.

**Lacunas no vault:**
- Nenhum ADR nativo em Obsidian.
- Sem template/checklist de onboarding para novos RPAs integrarem telemetria.
- Schema de `rpa_events` e `detalhes` não consolidado em uma nota — está espalhado entre módulos.
- Política de retenção registrada como backlog, sem decisão.
- Justificativa arquitetural de SSE vs WebSocket/gRPC só no ADR do repo.

---

## 6. Recomendação de contrato v1

Derivada do padrão que os 2 RPAs ativos já usam, resolvendo as divergências identificadas:

### Campos obrigatórios

| Campo | Tipo | Regra |
|-------|------|-------|
| `rpa` | string ≤100 | Identificador único **por RPA** (não por empresa). Cartorios passa a enviar `cartorios-<apelido>` |
| `event` | enum | `rpa_started` \| `rpa_finished` \| `rpa_error` \| `heartbeat` \| `automation_started` \| `automation_finished` |
| `session_id` | string ≤200 | UUID por execução |
| `machine` | string ≤200 | `socket.gethostname()` |
| `empresa` | string ≤200 | Contexto de negócio (cliente, empresa processada) |
| `timestamp` | string ISO 8601 UTC | `datetime.now(timezone.utc).isoformat()` |
| `contract_version` | string | `"v1"` — habilita migração futura lado servidor |

### Campos opcionais

| Campo | Tipo | Regra |
|-------|------|-------|
| `status` | enum | `success` \| `error` \| `warning` (validado com enum) |
| `duracao_segundos` | number | 0-86400, obrigatório em `*_finished` |
| `detalhes` | object | `{ records?, totalRecords?, description?, divergencias? }` — livre, persistido em `extra` JSONB |

### Auth

- Header `X-Api-Key` **obrigatório em `POST /events`**.
- Chave **por RPA** (não global), permitindo revogação granular.
- Enviar via env var `<RPA_MONITOR_API_KEY_ENV>` lida do `.env` ou `config.ini`.

### Heartbeat

- `event: "heartbeat"` a cada 60s durante execução longa (>2min).
- Permite dashboard mostrar "vivo" mesmo sem eventos de negócio.

### URL

- Passar a ler de env var `RPA_MONITOR_URL` em vez de hardcoded.

### Lib compartilhada

- Extrair `telemetry.py` para pacote Python interno (opções: git submodule, pip install -e, PyPI privado).
- Versionamento semântico acoplado ao `contract_version`.

---

## 7. Riscos e armadilhas

1. **Deploy coordenado de auth**: se `X-Api-Key` passar a ser realmente validada, os 2 RPAs ativos caem em 401 silencioso (POST assíncrono com thread daemon). Precisa rollout coordenado ou feature flag lado servidor.
2. **Renomear `rpa` → `rpa_name`**: quebra ambos os clientes. Melhor atualizar o contrato doc para refletir o código atual (`rpa`).
3. **Migração Cartorios de `cartorios` → `cartorios-<apelido>`**: parte histórica ficará com o nome antigo, visões agregadas partidas. Opções: (a) backfill em `rpa_events`, (b) manter view de compatibilidade no servidor, (c) aceitar corte na virada.
4. **Lib duplicada → pacote compartilhado**: transição exige alinhar versões Python, build do PyInstaller (imports dinâmicos podem quebrar `.spec`).
5. **Sem retenção**: decisão urgente antes de `rpa_events` crescer mais. Neon serverless cobra por storage.
6. **Thread daemon silencia falhas**: clientes não sabem se telemetria está chegando. Considerar modo debug com log visível em caso de erro de rede.
7. **`event` / `status` sem enum**: adicionar validação no servidor pode rejeitar eventos legados (ex: REBNIC com `automation_*` se não estiver na enum).
8. **ORG-AGENT endpoint paralelo**: `POST /api/telemetria/org-agent` está no backlog do vault e usa schema totalmente diferente (tokens, custo). Decidir se v1 cobre só RPAs ou também ORG-AGENT.

---

## 8. Perguntas abertas pro Davi

1. **Auth em prod**: o servidor em <hosting> está realmente exigindo `X-Api-Key` hoje? Se sim, como os 2 RPAs atuais estão reportando sem enviar o header? (Suspeita: `<RPA_API_KEY_ENV>` vazia em <hosting>, caindo no `if (!key ...)` → 401 silencioso não visto por ninguém.)
2. **Identificação do Cartorios**: migrar pra `rpa=cartorios-<apelido>` (aline, tiago, ...) ou manter `cartorios` + `empresa` e agregar por alias no servidor?
3. **Escopo do contrato v1**: cobre só os 2 RPAs que já reportam, ou já planeja onboarding dos 4 que não reportam (SITTAX, Capt-V1, Capt-V2)? FLUXO-CAIXA (Streamlit dashboard) entra?
4. **Lib compartilhada**: qual mecanismo de distribuição — git submodule, pip install -e num path relativo, PyPI privado, pacote no repo MONITOR-RPA?
5. **Heartbeat**: entra no v1 ou fica pro v2? Intervalo 30s/60s/120s?
6. **Server-Monitoracao**: posso marcar como deprecated oficialmente e remover do escopo, ou ainda serve de algo?
7. **ORG-AGENT** (`POST /api/telemetria/org-agent` mencionado no vault): entra no mesmo v1 ou é um contrato separado?
8. **Retenção**: qual política aceitável — 30d, 90d, 1 ano? Archive ou delete?
9. **Algum RPA que não capturei**: só olhei os 6 diretórios em `/Users/davicassoli/Trabalho/`. Tem mais algum RPA em outro host/repo que reporta (ou deveria reportar) telemetria?
