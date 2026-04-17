# SDD — `grupo14d-telemetry` v0.1

**Specification-Driven Development Document**
**Autor:** Davi Cassoli Lira
**Data:** 2026-04-17
**Status:** Draft para execução
**Escopo:** telemetria apenas; outros módulos (parsers, auth, conciliação) virão em libs irmãs depois

---

## 1. Contexto

### 1.1 Situação atual (evidência, não suposição)

Baseado no relatório de exploração v2 (`/tmp/exploracao-telemetry-v2.md`, 2026-04-17):

- 15 RPAs em produção, **100% reportando** telemetria ao MONITOR-RPA (`<MONITOR_INTERNAL_URL>`)
- Uma única geração de `telemetry.py` — **15 cópias idênticas** espalhadas pelos repos
- Auth via `X-Api-Key` funcionando em todos
- Heartbeat existe **apenas no RPA-IRPF** (3 watchers, 300s), payload já estruturado
- Servidor aceita `additionalProperties: true` no `POST /events` — schema permissivo
- Zero versionamento de contrato — qualquer evolução futura vai doer

### 1.2 Problemas a resolver

| # | Problema | Evidência |
|---|----------|-----------|
| P1 | 15 cópias da lib com drift garantido no tempo | Relatório v2, seção 3.4 |
| P2 | Sem `contract_version` — evolução cega | Relatório v2, seção 3.6 |
| P3 | Heartbeat só no IRPF — zona cega em 14 RPAs | Relatório v2, seção 3.5 |
| P4 | Naming inconsistente (`trots_alt_entrada`, `Rebnic-Acumuladores`) | Relatório v2, seção 3.1 |
| P5 | 3 formatos de config coexistindo (INI, YAML, config-global) | Relatório v2, seção 3.11 |

### 1.3 O que NÃO está no escopo do v0.1

- Auth por RPA (hoje é chave global — decisão adiada)
- Retenção de `rpa_events`
- Gate em endpoints admin abertos
- Enum strict no servidor (só documentamos, não impomos)
- Parsers, reconciliação, motor de regras (outras libs futuras)
- Migração do MONITOR-RPA em si

---

## 2. Objetivos e não-objetivos

### 2.1 Objetivos (o v0.1 tem sucesso se…)

1. **Única fonte de verdade**: `grupo14d-telemetry` é o único lugar onde lib de telemetria existe
2. **Adoção 15/15**: todos os RPAs passam a depender do pacote, zero cópias locais sobrevivem
3. **Contrato v1 documentado e versionado**: payload carrega `contract_version="v1"`
4. **Heartbeat universal**: qualquer RPA pode ativar heartbeat 60s com 1 linha de código
5. **Naming normalizado**: `trots_alt_entrada → trots-alt-entrada`, `Rebnic-Acumuladores → rebnic-acumuladores`, `trots_import_nfse_tomados → trots-import-nfse-tomados`
6. **Histórico preservado**: `rpa_events` recebe backfill dos nomes antigos para os novos
7. **Migração incremental**: RPAs migram 1 por 1, nenhum deploy grande explosivo
8. **Rollback trivial**: se v0.1 der problema, um RPA volta pra versão anterior só mexendo no `requirements.txt`

### 2.2 Não-objetivos explícitos

- Não vamos trocar o schema do servidor (`rpa_events`, `eventSchema`)
- Não vamos adicionar retry com fila persistente (thread daemon fica)
- Não vamos fazer CI/CD ainda
- Não vamos publicar em PyPI privado
- Não vamos cobrir 100% de testes — smoke tests suficientes

---

## 3. Especificação do contrato v1

### 3.1 Payload

**Campos obrigatórios:**

| Campo | Tipo | Regra | Exemplo |
|-------|------|-------|---------|
| `contract_version` | string | Literal `"v1"` | `"v1"` |
| `rpa` | string, ≤100 | kebab-case, `[a-z0-9-]+` | `"rebnic-acumuladores"` |
| `event` | string, ≤100 | Um dos eventos canônicos (ver 3.2) | `"rpa_started"` |
| `session_id` | string, ≤200 | UUID4 completo, não truncado | `"550e8400-e29b-41d4-a716-446655440000"` |
| `machine` | string, ≤200 | `socket.gethostname()` | `"DESKTOP-14D-03"` |
| `empresa` | string, ≤200 | Slug livre; convenção de negócio | `"empresa-x-ltda"` |
| `timestamp` | string ISO 8601 UTC | `datetime.now(timezone.utc).isoformat()` | `"2026-04-17T14:30:00.123456+00:00"` |

**Campos opcionais:**

| Campo | Tipo | Regra |
|-------|------|-------|
| `status` | enum | `success` \| `error` \| `warning` \| `running` |
| `duracao_segundos` | number | `[0, 86400]`, obrigatório em eventos `*_finished` |
| `detalhes` | object | Livre; convenções: `records`, `totalRecords`, `description`, `divergencias`, `last_file`, `errors_since_start` |

### 3.2 Eventos canônicos

| Evento | Quando emitir | `status` obrigatório? | `duracao_segundos` obrigatório? |
|--------|---------------|----------------------|----------------------------------|
| `rpa_started` | Início da execução do RPA | não | não |
| `rpa_finished` | Fim (sucesso ou falha controlada) | sim | sim |
| `rpa_error` | Exceção não tratada | não (implícito `error`) | não |
| `heartbeat` | Pulso periódico a cada 60s em execução contínua | não (implícito `running`) | não |
| `automation_started` | Início de sub-tarefa dentro do RPA | não | não |
| `automation_finished` | Fim de sub-tarefa | sim | sim |
| `watcher_started` | Boot de daemon/watcher | não | não |
| `watcher_heartbeat` | Pulso de watcher (alias de `heartbeat` pra retrocompatibilidade IRPF) | não | não |

### 3.3 Compatibilidade com servidor atual

Servidor (`MONITOR-RPA/apps/api/server.ts`) **não muda no v0.1**:
- `eventSchema` continua com `additionalProperties: true`
- Payloads sem `contract_version` continuam aceitos (tratados como `v0` implícito)
- Enum de `event`/`status` não é imposto server-side ainda
- Backfill de nomes é feito via SQL direto no Neon, não via API

---

## 4. API pública do pacote

### 4.1 Estrutura

```
grupo14d-telemetry/
├── pyproject.toml
├── README.md
├── CHANGELOG.md
├── .gitignore
├── src/
│   └── grupo14d_telemetry/
│       ├── __init__.py
│       ├── _version.py
│       ├── client.py          # Telemetry (classe principal)
│       ├── heartbeat.py       # HeartbeatDaemon
│       ├── config.py          # load_config (INI, YAML, env)
│       ├── events.py          # enum de eventos + validação leve
│       └── _internal.py       # helpers privados (UUID, hostname, etc)
├── tests/
│   └── test_smoke.py
└── examples/
    ├── minimal.py
    └── with_heartbeat.py
```

### 4.2 API pública (apenas estes símbolos são exportados)

```python
from grupo14d_telemetry import (
    Telemetry,           # classe principal
    HeartbeatDaemon,     # daemon de heartbeat
    TelemetryConfig,     # dataclass de config
    load_config,         # função de carregamento auto (INI/YAML/env)
    __version__,
)
```

### 4.3 Contrato da classe `Telemetry`

```python
class Telemetry:
    def __init__(
        self,
        rpa_name: str,
        server_url: str,
        api_key: str | None = None,
        enabled: bool = True,
        timeout: float = 5.0,
        session_id: str | None = None,  # default: uuid4()
    ) -> None: ...

    @classmethod
    def from_config(cls, config_path: str | None = None, **overrides) -> "Telemetry":
        """Carrega config.ini, pipeline.yml ou env. Overrides ganham."""

    # ---- eventos de ciclo de vida ----
    def start(self, empresa: str = "", detalhes: dict | None = None) -> None: ...
    def finish(self, status: str = "success", empresa: str = "", detalhes: dict | None = None) -> None: ...
    def error(self, message: str, empresa: str = "", detalhes: dict | None = None) -> None: ...

    # ---- eventos de sub-tarefa ----
    def automation_start(self, empresa: str = "", detalhes: dict | None = None) -> None: ...
    def automation_finish(self, status: str = "success", empresa: str = "", detalhes: dict | None = None) -> None: ...

    # ---- eventos de watcher/daemon ----
    def watcher_started(self, empresa: str = "", detalhes: dict | None = None) -> None: ...

    # ---- low-level ----
    def send(self, event: str, empresa: str = "", detalhes: dict | None = None, **extra) -> None:
        """Envia evento bruto. Usado internamente e por eventos custom."""
```

### 4.4 Contrato da classe `HeartbeatDaemon`

```python
class HeartbeatDaemon:
    def __init__(
        self,
        telemetry: Telemetry,
        empresa: str = "",
        interval_seconds: int = 60,
        detalhes_provider: Callable[[], dict] | None = None,
    ) -> None: ...

    def start(self) -> None: ...
    def stop(self) -> None: ...

    def __enter__(self) -> "HeartbeatDaemon": ...
    def __exit__(self, *args) -> None: ...
```

**Uso típico:**

```python
from grupo14d_telemetry import Telemetry, HeartbeatDaemon

tel = Telemetry.from_config()  # lê config.ini automaticamente

tel.start(empresa="empresa-x")
try:
    with HeartbeatDaemon(tel, empresa="empresa-x", interval_seconds=60):
        # RPA faz seu trabalho aqui — heartbeats saem em background
        processar_empresa()
    tel.finish(status="success", empresa="empresa-x")
except Exception as e:
    tel.error(str(e), empresa="empresa-x")
    raise
```

### 4.5 Resolução de config (ordem de precedência)

1. Argumentos explícitos no `Telemetry(...)` ou overrides em `from_config()`
2. Variáveis de ambiente: `<RPA_API_KEY_ENV>`, `MONITOR_RPA_URL`, `RPA_NAME`
3. Arquivo `config.ini` (seção `[telemetry]`)
4. Arquivo `config-global.ini` (seção `[telemetry]`) — compat Cartorios
5. Arquivo `config/pipeline.yml` (bloco `telemetry:`) — compat TROTS-IMPORT
6. Arquivos de chave: `<api-key-file>`, `api_key.txt`, `<secrets-api-key-file>`

**Primeiro achado vence.** Logs em nível DEBUG indicam qual fonte foi usada.

### 4.6 Comportamento de rede

- Timeout: 5s
- Thread daemon, fire-and-forget
- Falha de rede: log DEBUG + swallow (NUNCA derruba o RPA)
- Sem retry exponencial no v0.1 (mantém simplicidade da geração atual)
- Sem fila persistente (se a rede cair, eventos são perdidos — trade-off explícito)

---

## 5. Critérios de aceite (verificáveis)

### 5.1 Pacote

- [ ] `pip install -e .` funciona em venv limpo (Python 3.11+)
- [ ] `python -c "from grupo14d_telemetry import Telemetry, HeartbeatDaemon, __version__; print(__version__)"` imprime `0.1.0`
- [ ] README tem exemplo copy-paste executável
- [ ] Exemplo `minimal.py` roda e reporta 1 evento ao MONITOR em dev

### 5.2 Contrato v1

- [ ] Todo evento enviado pelo pacote inclui `contract_version="v1"`
- [ ] `rpa_name` sempre é kebab-case (validado no `__init__`, raise `ValueError` se não bater regex `^[a-z0-9][a-z0-9-]*$`)
- [ ] `timestamp` sempre é UTC com timezone (`datetime.now(timezone.utc).isoformat()`)
- [ ] `session_id` é UUID4 completo

### 5.3 Servidor (MONITOR-RPA)

- [ ] MONITOR-RPA continua aceitando payloads sem `contract_version` (backcompat)
- [ ] MONITOR-RPA não é alterado no v0.1 (apenas SQL de backfill)

### 5.4 Backfill de nomes

- [ ] Script SQL idempotente em `scripts/backfill-v1-naming.sql`
- [ ] Renomeia em `rpa_events.rpa`:
  - `trots_alt_entrada` → `trots-alt-entrada`
  - `trots_import_nfse_tomados` → `trots-import-nfse-tomados`
  - `Rebnic-Acumuladores` → `rebnic-acumuladores`
- [ ] Renomeia em `rpa_config.rpa` se aplicável
- [ ] Dry-run mode: imprime contagem de linhas afetadas antes de aplicar
- [ ] Registrado em nota de vault como ADR

### 5.5 Migração dos 15 RPAs

- [ ] Ordem definida na seção 6.2
- [ ] RPA-piloto migrado, rodando em produção sem regressão por ≥3 dias antes de migrar o próximo
- [ ] Cada RPA migrado tem commit `chore(telemetry): migrar para grupo14d-telemetry vX.Y.Z`
- [ ] `telemetry.py` local é **removido** (não comentado, não deprecated — removido)
- [ ] `requirements.txt` pinna versão específica: `grupo14d-telemetry @ git+ssh://...@v0.1.0`

---

## 6. Plano de execução

### 6.1 Sprints (6 sprints, ~1-2 dias cada)

#### Sprint A — Repo e scaffolding (meio dia)

**Pré-requisitos:** repo `grupo14d-telemetry` criado no GitHub.

1. Estrutura de pastas conforme 4.1
2. `pyproject.toml` com Python ≥3.11, deps: nenhuma externa (usar `urllib` da stdlib pra não criar nova dep de produção)
3. `_version.py` → `__version__ = "0.1.0.dev0"`
4. `__init__.py` exporta os 5 símbolos públicos como placeholders
5. `.gitignore`, README mínimo, CHANGELOG com `[Unreleased]`
6. Commit: `chore: scaffolding`

**Pronto quando:** `pip install -e .` funciona, `from grupo14d_telemetry import *` não quebra.

#### Sprint B — Port da lib atual + contract_version (1 dia)

1. Copiar `telemetry.py` de um RPA (sugestão: **RPA-IRPF** — já tem heartbeat, é a referência)
2. Refatorar pra módulos separados: `client.py`, `heartbeat.py`, `config.py`, `events.py`, `_internal.py`
3. Adicionar `contract_version="v1"` em todo payload enviado (no método `send()`)
4. Adicionar validação de `rpa_name` kebab-case no `__init__` (regex)
5. Adicionar enum de `event` em `events.py` (só validação preguiçosa — warn, não raise)
6. Escrever 3 smoke tests: start/finish/error ponta a ponta contra servidor fake local
7. Commit: `feat: porta telemetry.py e adiciona contract_version`

**Pronto quando:** um script de 15 linhas reporta start + finish + erro pro MONITOR dev e tudo chega com `contract_version="v1"`.

#### Sprint C — HeartbeatDaemon generalizado (1 dia)

1. Extrair lógica de `RPA-IRPF/src/watcher/pdf_watcher.py:118-140` pra `heartbeat.py`
2. Generalizar: aceitar `detalhes_provider` callable que o RPA passa pra customizar o payload
3. Interval default 60s (decisão tomada)
4. Context manager (`__enter__`/`__exit__`)
5. `threading.Event.wait()` em vez de `time.sleep()` pra stop ser instantâneo
6. Smoke test: 3 minutos de loop, conta heartbeats recebidos no servidor fake
7. Commit: `feat: HeartbeatDaemon generalizado 60s`

**Pronto quando:** exemplo `with_heartbeat.py` roda 2 min e servidor fake recebe 2 heartbeats.

#### Sprint D — Config loader multi-formato (meio dia)

1. Implementar `load_config()` com cascata da seção 4.5
2. Suportar `.ini`, `.yml`, env vars, arquivos de chave
3. Log DEBUG informando qual fonte foi usada
4. `Telemetry.from_config()` como método de conveniência
5. Commit: `feat: config loader INI/YAML/env`

**Pronto quando:** os 3 formatos que existem hoje (INI, config-global.ini, pipeline.yml) são carregados sem erro.

#### Sprint E — Release v0.1.0 + backfill SQL (meio dia)

1. Bump `_version.py` para `0.1.0`
2. CHANGELOG consolidado
3. Tag `v0.1.0`, push
4. Testar `pip install git+ssh://git@github.com/A-DAVI/grupo14d-telemetry.git@v0.1.0` em venv novo
5. Escrever `scripts/backfill-v1-naming.sql` com dry-run e apply modes
6. Rodar dry-run em prod — validar contagens
7. Rodar apply em prod (fora de horário comercial)
8. Nota no vault: `grupo14d-telemetry/v0.1.0-release.md`
9. Commit: `chore: release v0.1.0`

**Pronto quando:** tag criada, backfill aplicado, nota de release no vault.

#### Sprint F — Migração dos 15 RPAs (2-3 semanas, RPA por RPA)

Ver 6.2.

### 6.2 Ordem de migração dos RPAs

Critério: **menor risco primeiro, ganha confiança, escala.**

| Ordem | RPA | Razão |
|-------|-----|-------|
| 1 | **RPA-SISTEMA-GRUPO14D** | Piloto. Baixo tráfego, passou a reportar hoje, bug aqui não afeta fiscal |
| 2 | **RPA-BALANCETES** | Similar ao 1, baixo risco |
| 3 | **RPA-DROGARIA** | Idem |
| 4 | **RPA-TROTS** | Idem |
| 5 | **RPA-TROTS-ALT-DE-ENTRADA** | Aqui já migra naming `snake → kebab` |
| 6 | **RPA-TROTS-IMPORT** | Idem + valida loader YAML |
| 7 | **RPA-Cartorios** | 6 cartórios, valida reuso de 1 RPA em múltiplas empresas |
| 8 | **RPA-Conferencia-NFs-e** | Fiscal, volume médio |
| 9 | **RPA-COBRANCA-AUTOMATICA** | Crítico pra receita; só migra após 8 RPAs estáveis |
| 10 | **RPA-SITTAX** | Idem |
| 11 | **RPA-XML-HOSPITAL** | Idem |
| 12 | **RPA-LANCAMENTOS-REBNIC** | Idem |
| 13 | **RPA-REBNIC** | Idem |
| 14 | **RPA-REBNIC-ALTERACAO-ACUMULADOR-DOMINIO** | Valida eventos `automation_*` + rename PascalCase |
| 15 | **RPA-IRPF** | **Por último.** É o mais crítico, mais complexo (3 watchers), e foi a fonte do port. Última coisa a trocar. |

**Regra de ouro:** 3 dias úteis de observação por RPA migrado antes de migrar o próximo. Se qualquer métrica (volume de eventos, taxa de erro, heartbeats faltando) desviar, pausa e investiga.

### 6.3 Procedimento por RPA (runbook)

1. Branch `feat/migrate-to-grupo14d-telemetry`
2. `pip install grupo14d-telemetry @ git+ssh://...@v0.1.0` em `requirements.txt`
3. Substituir `from telemetry import Telemetry` por `from grupo14d_telemetry import Telemetry`
4. Se o `rpa_name` estiver em snake ou PascalCase, trocar pra kebab **e confirmar que backfill já rodou**
5. **Remover** `telemetry.py` local do repo
6. Rodar smoke em dev contra MONITOR de dev
7. Merge, deploy, observar 3 dias
8. Commit: `chore(telemetry): migrar para grupo14d-telemetry v0.1.0`

---

## 7. Riscos e mitigações

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|-------|---------------|---------|-----------|
| R1 | Backfill SQL corrompe `rpa_events` | Baixa | Alto | Dry-run obrigatório + backup do Neon antes do apply |
| R2 | Lib nova tem bug que silenciosa | Média | Alto | RPA-piloto com baixo risco, 3 dias de observação antes do próximo |
| R3 | Heartbeat 60s sobrecarrega Neon | Baixa | Médio | Medir após 3 RPAs migrados com heartbeat ativo; recuar pra 120s se necessário |
| R4 | Pip install via SSH quebra em máquina de produção (sem chave) | Média | Médio | Documentar setup de deploy key no runbook; fallback HTTPS com token |
| R5 | PyInstaller não empacota lib corretamente | Média | Alto | Testar `.spec` já no RPA-piloto; usar `--collect-all grupo14d_telemetry` se preciso |
| R6 | RPA em produção fica sem telemetria durante janela de migração | Baixa | Baixo | Migração é hot-swap; timeout 5s não trava |
| R7 | `contract_version` quebra algum parser downstream | Baixa | Baixo | Servidor tá com `additionalProperties: true`; campo novo é inócuo |

---

## 8. Decisões registradas (ADRs embutidas)

### ADR-01: Pacote separado vs monorepo `grupo14d-toolkit`

**Decisão:** repo próprio `grupo14d-telemetry`.

**Razão:** Telemetria é dep horizontal — todos os RPAs dependem. Criar `grupo14d-toolkit` como monorepo agora obrigaria cada RPA a baixar parsers, auth, etc. que não usam. Futuros módulos (parsers, auth) nascem como repos próprios também. Se um dia virar volume, consolida em toolkit num `v1.0`.

### ADR-02: Sem retry persistente

**Decisão:** mantém thread daemon fire-and-forget, timeout 5s, log DEBUG em falha.

**Razão:** Comportamento atual funciona. Adicionar fila persistente (SQLite local) dobra a complexidade da lib pra cobrir caso raro (rede cair no último segundo). Revisitar em v0.2 se perda de evento virar dor real.

### ADR-03: Heartbeat 60s

**Decisão:** default 60s.

**Razão:** Davi escolheu granularidade sobre economia de storage. Se custo no Neon subir demais após rollout, `HeartbeatDaemon(interval_seconds=N)` permite ajuste por RPA sem mexer na lib.

### ADR-04: Backfill com preservação de histórico

**Decisão:** `UPDATE rpa_events SET rpa = 'trots-alt-entrada' WHERE rpa = 'trots_alt_entrada'` (etc).

**Razão:** Histórico importa pra analytics (`business-analytics` endpoint usa série temporal). Corte seco perderia continuidade visual no dashboard.

### ADR-05: `contract_version` obrigatório no cliente, opcional no servidor

**Decisão:** cliente sempre envia `"v1"`, servidor aceita ausência como `"v0"` implícito.

**Razão:** Migração gradual dos 15 RPAs significa que, durante semanas, vão coexistir payloads v0 (sem versão) e v1 (com versão). Servidor tem que aceitar os dois. Quando 15/15 estiverem em v1, aí sim pode virar obrigatório no servidor (v0.2).

---

## 9. Perguntas ainda abertas (não bloqueantes pro v0.1)

Estas foram identificadas no relatório v2 mas ficam registradas pra decisão futura. Minha recomendação em cada uma:

| # | Questão | Minha recomendação |
|---|---------|---------------------|
| Q1 | Chave API por RPA (vs global) | v0.2 — só faz sentido quando houver revogação granular real |
| Q2 | Cartorios como 1 RPA ou 6? | Manter como 1 agora, `empresa` basta. Reavaliar se o dashboard pedir |
| Q3 | Enum strict server-side | v0.2 — depois que 15/15 estiverem em v1 |
| Q4 | Retenção de `rpa_events` | Criticar nos próximos 30 dias se volume escalar; sugestão 180 dias |
| Q5 | Gate em endpoints admin | v0.2 — mas **urgente** se servidor sair da rede interna |
| Q6 | Desligar deploy <hosting> | Desligar já; é desperdício de recurso |

---

## 10. Critério de sucesso global

O v0.1 é considerado entregue quando:

- [ ] `grupo14d-telemetry v0.1.0` está publicado como tag Git
- [ ] Backfill SQL aplicado em prod com validação
- [ ] 15/15 RPAs migrados, `telemetry.py` local removido de cada repo
- [ ] Heartbeat 60s ativo em pelo menos 3 RPAs de execução contínua
- [ ] Zero incidente de produção atribuível à migração
- [ ] Nota de release no vault, com link pro SDD e pros ADRs

---

## 11. Próximos passos imediatos

1. Você revisa este SDD, aponta discordâncias
2. Eu reescrevo o `PLANO_TOOLKIT_V0.1.md` antigo como `PROMPT_CLAUDE_CODE_SPRINT_A.md` (só Sprint A, pra começar)
3. Você cria o repo `grupo14d-telemetry` no GitHub (vazio)
4. Dispara o prompt do Sprint A no Claude Code
5. Iteramos Sprint B em diante conforme os entregáveis saem
