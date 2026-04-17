# P3 — Avaliação do SDD `grupo14d-telemetry` v0.1

**Avaliador:** Claude (baseado em P2-V2 + análise crítica)  
**Data:** 2026-04-17  
**Status:** Viável com ajustes críticos  
**Tempo estimado para sprint A-E:** 1-2 semanas; Migração F: 3-4 semanas  

---

## 0. Resumo executivo

O SDD é **bem pensado e exequível**, mas tem **3 críticas bloqueantes + 7 observações** que precisam resolução antes de disparar código. A maior delas: o **backfill de nomes em produção é arriscado** e precisa workflow muito mais conservador. O resto é ajuste fino.

| Categoria | Status | Ação |
|-----------|--------|------|
| Especificação de contrato v1 | ✅ Sólida | Usar como-é |
| Estrutura do pacote | ✅ Razoável | Minor tweak em `_internal.py` |
| Sprints A-E | ✅ Factível | Cronograma ok, nada novo a inventar |
| Ordem de migração (Sprint F) | ⚠️ Arriscada | Reordenar IRPF pra mais cedo; adicionar smoke test obrigatório entre RPAs |
| Backfill SQL | 🔴 **CRÍTICO** | Reescrever com canary + dry-run 3× antes de apply |
| Heartbeat 60s | ⚠️ Questionável | Medir custo no Neon em dev; pronto pra recuar pra 300s |
| ADRs | ✅ Claras | Bem justificadas; PDR-02 revisitar em v0.2 |

---

## 1. Críticas bloqueantes (resolver antes de começar)

### 1.1 🔴 Backfill de nomes — risco de anomalia histórica

**Problema:**

O SDD propõe:
```sql
UPDATE rpa_events SET rpa = 'trots-alt-entrada' WHERE rpa = 'trots_alt_entrada'
```

Isto é **simples demais**. No mundo real, quando fazemos isso em produção com 500k+ linhas de histórico, precisamos de:

1. **Validação prévia**: contar linhas afetadas, verificar se nenhum nome novo (`trots-alt-entrada`) já existe.
2. **Reversibilidade**: backup full do Neon ou stored procedure com rollback.
3. **Observação pós-apply**: 2-4h monitorando se o `POST /events` continua funcionando (índices estão ok? Queries antigas com o nome antigo agora quebram?).
4. **Comunicação**: avisar ops se houver qualquer ferramenta/dashboard que hardcode o nome antigo.

**Recomendação:**

```sql
-- 1. VALIDAÇÃO (rodar isolado pra inspecionar)
SELECT COUNT(*) as affected_count FROM rpa_events 
WHERE rpa IN ('trots_alt_entrada', 'trots_import_nfse_tomados', 'Rebnic-Acumuladores');

-- 2. BACKUP (criar tabela de histórico)
CREATE TABLE rpa_events_backup_v1_naming AS 
SELECT * FROM rpa_events 
WHERE rpa IN ('trots_alt_entrada', 'trots_import_nfse_tomados', 'Rebnic-Acumuladores');

-- 3. DRY-RUN (criar update statement, NÃO executar)
BEGIN;
  UPDATE rpa_events SET rpa = 'trots-alt-entrada' WHERE rpa = 'trots_alt_entrada';
  UPDATE rpa_events SET rpa = 'trots-import-nfse-tomados' WHERE rpa = 'trots_import_nfse_tomados';
  UPDATE rpa_events SET rpa = 'rebnic-acumuladores' WHERE rpa = 'Rebnic-Acumuladores';
ROLLBACK; -- NÃO APLICA, SÓ MOSTRA ERROS SE HOUVER

-- 4. APPLY (com timeout longo)
-- Executar fora de horário comercial, com ops observando
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
  -- updates aqui
COMMIT;
```

Adicionar isto ao Sprint E e marcar como "rodou com sucesso" antes de migrar qualquer RPA.

---

### 1.2 🔴 Ordem de migração — IRPF muito tarde

**Problema:**

O SDD coloca IRPF em **último lugar (posição 15)**:

> "**Por último.** É o mais crítico, mais complexo (3 watchers), e foi a fonte do port."

Isto está **errado** por 3 razões:

1. **Foi a fonte do port** → código está fresco, bugs saem na Sprint B, não na Sprint F.
2. **3 watchers = ótimo teste** → se HeartbeatDaemon quebra, IRPF é o indicador mais óbvio (10+ heartbeats/min durante teste). Descobrir cedo.
3. **Criticidade ≠ tardio** → ao contrário: colocar o mais crítico *cedo* com observação longa (1-2 semanas) dá confiança. Deixar pro fim é arriscado.
4. **Riscos de v0.1 aparecem no IRPF** → se houver memory leak em HeartbeatDaemon, vai ficar visível em thread daemon rodando 24/7.

**Recomendação:**

Mover IRPF para **posição 2 ou 3** (após RPA-SISTEMA-GRUPO14D piloto):

```
1. RPA-SISTEMA-GRUPO14D (piloto, 3 dias)
2. RPA-IRPF (referência do port, 7-10 dias observação longa)
3. RPA-BALANCETES
4. RPA-DROGARIA
5. RPA-TROTS
... (resto em ordem original)
```

Justificativa: IRPF migra cedo, roda 2 semanas limpo, ganha confiança pra escalar. Se quebra, ciclo de feedback é mínimo.

---

### 1.3 🔴 Smoke test obrigatório entre RPAs

**Problema:**

O SDD descreve mitigação:

> "RPA-piloto com baixo risco, 3 dias de observação antes do próximo"

Mas **não define o que é "observação"**. Sem critério explícito, é subjetivo. Alguém pode dizer "3 dias passaram, próximo!" sem data real nem métricas.

**Recomendação:**

Adicionar **SLA de migração** antes de liberar o próximo RPA:

```
Checklist pós-migração (deve estar 100% antes de desbloquear próximo):
- [ ] Mínimo 100 eventos do RPA novo reportados com contract_version="v1"
- [ ] Nenhum 5xx no MONITOR em /api/status 
- [ ] Heartbeat (se ativo) chegando a cada 60±5s (tolerância 5s)
- [ ] Nenhuma regressão em RTT de POST /events (< 200ms p99)
- [ ] Logs do RPA: zero "telemetry error" ou "connection timeout" ao MONITOR
- [ ] Dashboard visual: o RPA novo aparece em /api/stream e /api/status sem lag
```

Sugerir integração com Grafana ou simples script de curl que valida isto.

---

## 2. Observações importantes (não-bloqueantes, mas críticas)

### 2.1 ⚠️ Config loader — ordem de precedência pode ter hiato

**Observação:**

Seção 4.5 lista 6 níveis de cascata. Mas faltou: **MONITOR_RPA_URL vs RPA_NAME vs SERVER_URL**. RPAs atuais usam `config.ini [telemetry]` com `server_url` (lowercase). SDD menciona env `MONITOR_RPA_URL`. Qual é a fonte de verdade?

**Recomendação:**

Ir ao spec e decidir:

```python
# Sugestão: env > config.ini > default
server_url = (
    os.environ.get('MONITOR_RPA_URL')  # env (novo, para override rápido)
    or config.get('telemetry', 'server_url')  # config.ini (atual, padrão)
    or '<MONITOR_INTERNAL_URL>'  # hardcoded default
)
```

E documentar isto explicitamente em `config.py` docstring.

---

### 2.2 ⚠️ Logging — nível DEBUG pode não ser suficiente

**Observação:**

SDD diz:

> "Log DEBUG indicando qual fonte foi usada"

Isto é bom pra dev. Mas **em produção**, quando um RPA fica em modo silencioso (falha na rede sem crash), logs DEBUG ficam invisíveis. A thread daemon vai silenciosamente retrying... ou não retrying.

**Recomendação:**

Adicionar enum de "health" e permitir que o RPA consulte:

```python
class Telemetry:
    @property
    def is_healthy(self) -> bool:
        """Se False, última 5 mensagens falharam. Idealmente retrying or log-heavy."""
        ...
    
    @property
    def last_error(self) -> str | None:
        """Mensagem do último erro de rede (se houver)."""
        ...
```

Assim, um RPA crítico pode fazer:

```python
if not telemetry.is_healthy:
    logger.critical(f"TELEMETRY OFFLINE: {telemetry.last_error}")
    # decide se continua ou não
```

---

### 2.3 ⚠️ Heartbeat 60s — custo no Neon não foi validado

**Observação:**

SDD assume 60s é ok:

> "Davi escolheu granularidade sobre economia de storage."

Mas não há baseline. Se todos os 15 RPAs mandarem heartbeat a cada 60s durante 8h/dia = 720 eventos/dia × 15 RPAs = 10.8k eventos/dia.

Isto é negligenciável em volume. **Mas se algum RPA fica com heartbeat em thread daemon 24/7**, a conta muda:
- 1 RPA × 1440 heartbeats/dia × 365 dias = ~500k registros/ano por RPA.

Neon cobra por storage. Sem janela de retenção, isto cresce.

**Recomendação:**

Adicionar ao Sprint B:
1. Teste de carga: 10 HeartbeatDaemon paralelos por 1h, medir ingestão.
2. Registrar linha de base: "X bytes/dia por heartbeat".
3. Adicionar aviso em docstring se heartbeat ativo 24/7.

---

### 2.4 ⚠️ Validação de `rpa_name` kebab-case — regex pode ser mais apertada

**Observação:**

Seção 5.2 diz:

> "raise `ValueError` se não bater regex `^[a-z0-9][a-z0-9-]*$`"

Isto permite qualquer coisa. Melhor ser mais restritivo:

```python
^[a-z0-9]+(-[a-z0-9]+)*$  # força: começa letra/número, hífens no meio, termina letra/número
```

Exemplos:
- ✅ `rpa-irpf`, `rebnic-acumuladores`, `cartorios`, `balancete`
- ❌ `rpa-`, `-rpa`, `rpa--irpf` (duplos)

**Recomendação:**

Usar regex mais apertado:

```python
if not re.match(r'^[a-z0-9]+(-[a-z0-9]+)*$', rpa_name):
    raise ValueError(f"rpa_name deve ser kebab-case: {rpa_name}")
```

---

### 2.5 ⚠️ Thread daemon sem signal handler

**Observação:**

SDD não menciona sinal SIGTERM. Se um RPA for enviado `kill -15`, a thread daemon de heartbeat:

```python
while True:
    time.sleep(60)
    telemetry._send("heartbeat")
```

Continua rodando 60s a mais antes de perceber a morte da thread principal. É um incômodo menor, mas não é elegante.

**Recomendação:**

Usar `threading.Event` em vez de `time.sleep` (já sugerido em Sprint C, linha 323):

```python
def _heartbeat_loop(self):
    while not self._stop_event.is_set():
        self._stop_event.wait(timeout=60)  # espera 60s OU é sinalizado
        if not self._stop_event.is_set():
            self.telemetry._send("heartbeat")
```

Assim, `stop()` é instantâneo.

---

### 2.6 ⚠️ Sem testes unitários — apenas smoke tests

**Observação:**

SDD diz:

> "Não vamos cobrir 100% de testes — smoke tests suficientes"

Ok para v0.1. Mas após Sprint B, haveria valor em:

```
tests/
├── test_config_loader.py      # validar cascata de config
├── test_telemetry_payload.py  # validar schema v1
├── test_heartbeat_timing.py   # validar que intervals são respeitados
```

Estes pegam bugs cedo (ex: timezone, formato UUID) sem PyInstaller ou rede envolvida.

**Recomendação:**

Adicionar ao Sprint B: "adicione 3 testes unitários mínimos". Não é 100% coverage, mas captura os cases feliz mais o triste (`api_key=None`, `rpa_name="INVALID"`, etc).

---

### 2.7 ⚠️ PyInstaller — `.spec` é RPA-específico

**Observação:**

SDD Seção 7 (riscos) menciona:

> "PyInstaller não empacota lib corretamente" — Mitigação: "Testar `.spec` já no RPA-piloto"

Isto é ok, mas cada RPA pode ter `.spec` diferente. Alguns usam hooks customizados. Se a lib nova precisa de `--collect-all grupo14d_telemetry`, **isso muda em cada repo**.

**Recomendação:**

Adicionar exemplo `.spec` template no repo `grupo14d-telemetry`:

```
grupo14d-telemetry/examples/pyinstaller.spec
```

Com comentário:

```python
# Copie isto pra seu RPA e ajuste paths conforme necessário
a = Analysis(
    ['main.py'],
    ...
    collect_all_on_hidden_imports=['grupo14d_telemetry'],
)
```

---

## 3. Validações contra estado real (P2-V2)

### ✅ Especificação de contrato v1

**Validação:** Campos no SDD coincidem com payload atual dos RPAs.

- `contract_version` (novo) ✅ será adicionado
- `rpa` ✅ (hoje está em uso)
- `event` ✅ (rpa_started, rpa_finished, rpa_error já existem)
- `session_id` ✅ (uuid4, 200 chars)
- `machine` ✅ (hostname)
- `empresa` ✅ (já em uso)
- `timestamp` ✅ (ISO 8601 UTC — RPA-IRPF já usa)
- `status`, `duracao_segundos`, `detalhes` ✅ opcionais

**Resultado:** ✅ Especificação é compatível com estado real.

---

### ✅ Eventos canônicos

**Validação:** Mapeamento de eventos contra o que existe hoje.

| Evento | Existe hoje? | Notas |
|--------|-------------|-------|
| `rpa_started` | ✅ Sim | IRPF, Cartorios, REBNIC, etc |
| `rpa_finished` | ✅ Sim | Todos que usam `telemetry.finish()` |
| `rpa_error` | ✅ Sim | Todos que usam `telemetry.error()` |
| `automation_started` | ✅ Sim | REBNIC-Acumuladores (não documentado) |
| `automation_finished` | ✅ Sim | REBNIC-Acumuladores |
| `heartbeat` | ❌ Não — é `watcher_heartbeat` | IRPF usa nome específico; v1 formaliza como `heartbeat` |
| `watcher_started` | ⚠️ Parcial | IRPF usa em pdf_watcher.py, não documentado |
| `watcher_heartbeat` | ✅ Sim | IRPF (300s), será 60s em v1 |

**Resultado:** ✅ Todos os eventos v1 mapeiam para código existente. Mudança: intervalo de heartbeat (300s→60s em padrão, customizável).

---

### ⚠️ 15 RPAs reportando — confirmação numérica

**Validação:** SDD assume 15/15. P2-V2 também confirma.

Inventário P2-V2:
1. RPA-BALANCETES ✅
2. RPA-COBRANCA-AUTOMATICA ✅
3. RPA-Cartorios ✅ (reativado)
4. RPA-Conferencia-NFs-e ✅
5. RPA-DROGARIA ✅
6. RPA-IRPF ✅
7. RPA-LANCAMENTOS-REBNIC ✅
8. RPA-REBNIC ✅
9. RPA-REBNIC-ALTERACAO-ACUMULADOR-DOMINIO ✅
10. RPA-SISTEMA-GRUPO14D ✅
11. RPA-SITTAX ✅
12. RPA-TROTS ✅
13. RPA-TROTS-ALT-DE-ENTRADA ✅
14. RPA-TROTS-IMPORT ✅
15. RPA-XML-HOSPITAL ✅

**Ausente:** RPA-CAIXA-E-LUCROS (removed). ✅ Alinha com SDD.

**Resultado:** ✅ Contagem confirmada.

---

### ✅ URL única `<MONITOR_INTERNAL_URL>`

**Validação:** SDD assume URL única. P2-V2 confirma que <hosting> foi abandonado.

**Resultado:** ✅ Suposição válida.

---

### ⚠️ Nomenclatura kebab-case — 3 RPAs fora do padrão hoje

**Validação:** SDD quer forçar kebab-case. Hoje existem:

| RPA | Nomenclatura hoje | Esperado em v1 | Impacto |
|-----|-------------------|-----------------|---------|
| TROTS-ALT-DE-ENTRADA | `trots_alt_entrada` | `trots-alt-entrada` | Alta (mudar histórico) |
| TROTS-IMPORT | `trots_import_nfse_tomados` | `trots-import-nfse-tomados` | Alta |
| REBNIC-ACUMULADORES | `Rebnic-Acumuladores` | `rebnic-acumuladores` | Alta |

**Resultado:** ⚠️ Backfill é necessário e arriscado (ver crítica 1.1).

---

## 4. Factibilidade de sprints A-E

### ✅ Sprint A — Scaffolding (meio dia)

**Checklist:**
- [ ] Criar repo `grupo14d-telemetry` (pré-requisito)
- [ ] Estrutura do pyproject.toml → straightforward
- [ ] `_version.py` e `__init__.py` → 30 min
- [ ] README mínimo → 20 min
- [ ] `.gitignore` + CHANGELOG → 10 min

**Risco:** Baixo. É só boilerplate.

**Resultado:** ✅ **Factível, tempo realista.**

---

### ✅ Sprint B — Port da lib + contract_version (1 dia)

**Checklist:**
- [ ] Copiar `telemetry.py` de RPA-IRPF → 15 min refactor
- [ ] Dividir em módulos → 30 min
- [ ] Adicionar `contract_version="v1"` → 10 min
- [ ] Validação kebab-case → 20 min
- [ ] Enum de eventos (leve) → 30 min
- [ ] 3 smoke tests → 1.5h

**Risco:** Médio. Refactor pode ter surpresas (imports, paths). Mas o código já existe, é só reorganizar.

**Resultado:** ⚠️ **Factível, mas 1 dia é apertado. Planejar 1.5 dias.**

---

### ✅ Sprint C — HeartbeatDaemon (1 dia)

**Checklist:**
- [ ] Extrair lógica de IRPF → 30 min
- [ ] Generalizar com `detalhes_provider` → 1h
- [ ] Context manager + Event-based stop → 30 min
- [ ] Smoke test → 1h

**Risco:** Baixo. Lógica é simples e já existe em IRPF.

**Resultado:** ✅ **Factível, 1 dia ok.**

---

### ✅ Sprint D — Config loader multi-formato (meio dia)

**Checklist:**
- [ ] Suporte INI → 30 min (usar stdlib `configparser`)
- [ ] Suporte YAML → 30 min (usar `pyyaml`, 1 dep nova)
- [ ] Suporte env var → 10 min
- [ ] Log debug → 10 min
- [ ] `from_config()` classmethod → 10 min

**Risco:** Muito baixo. Operações triviais.

**Resultado:** ✅ **Factível, tempo realista.**

---

### ✅ Sprint E — Release + backfill (meio dia + risco)

**Checklist:**
- [ ] Bump `_version.py` a `0.1.0` → 2 min
- [ ] CHANGELOG consolidado → 30 min
- [ ] Tag e push → 5 min
- [ ] Teste pip install nova → 10 min
- [ ] **Script SQL com dry-run** → 1h (⚠️ tempo subestimado)
- [ ] Dry-run em produção → 30 min
- [ ] **Apply em produção** → 30 min + observação pós (1-2h)

**Risco:** Alto. Backfill em produção é crítico (ver crítica 1.1).

**Resultado:** ⚠️ **Factível, MAS marcar 2-3h de observação pós-apply como obrigatório.**

---

### ⚠️ Sprint F — Migração dos 15 RPAs (2-3 semanas)

**Checklist por RPA:**
- Branch + require `grupo14d-telemetry`
- Ajustar import
- Remover `telemetry.py` local
- Smoke test em dev
- Merge + deploy
- Observação 3+ dias (ou checklist da seção 2.1 desta avaliação)

**Risco:** Médio. Cada RPA é independente, mas 15 migrações seriais acumulam tempo. Se alguma der problema, ciclo de feedback é longo.

**Resultado:** ⚠️ **Factível, mas considerar paralelizar 2-3 RPAs de baixo risco em paralelo após IRPF pass.**

---

## 5. Cronograma revisto

Baseado em análise acima:

```
Sprint A (scaffolding):           meio dia         (2026-04-18)
Sprint B (port + contract):       1.5 dias         (2026-04-18 PM + 2026-04-19)
Sprint C (heartbeat):             1 dia            (2026-04-20)
Sprint D (config):                meio dia         (2026-04-20 PM)
Sprint E (release + backfill):    0.5 dias + 2h obs (2026-04-21 PM + 2026-04-22 obs)

Sprint F (migração):
  - Canary (RPA-SISTEMA, IRPF):   2 semanas        (2026-04-22 — 2026-05-06)
  - Batch 2 (RPA-BALANCETES…):    1 semana         (2026-05-06 — 2026-05-13)
  - Batch 3 (críticos):           1 semana         (2026-05-13 — 2026-05-20)

Total: ~3.5 semanas até 15/15 migrados
```

---

## 6. Recomendações (síntese)

| # | Crítica | Prioridade | Ação |
|----|---------|-----------|------|
| C1 | Backfill SQL muito ingênuo | 🔴 BLOQUEANTE | Reescrever com backup + dry-run 3× |
| C2 | IRPF em último lugar | 🔴 BLOQUEANTE | Mover para posição 2 ou 3 |
| C3 | Sem SLA de migração | 🔴 BLOQUEANTE | Adicionar checklist de 6 itens (seção 2.1) |
| O1 | Config loader ambígua | ⚠️ Importante | Documentar precedência de env vs config.ini |
| O2 | Logging nivel DEBUG | ⚠️ Importante | Adicionar propriedades `is_healthy` e `last_error` |
| O3 | Heartbeat 60s sem validação | ⚠️ Importante | Teste de carga em Sprint B, anotar linha de base |
| O4 | Regex `rpa_name` loose | ⚠️ Menor | Usar `^[a-z0-9]+(-[a-z0-9]+)*$` |
| O5 | Signal handler SIGTERM | ⚠️ Menor | Usar `threading.Event` em HeartbeatDaemon |
| O6 | Sem testes unitários | ⚠️ Menor | Adicionar 3 testes (config, payload, heartbeat timing) |
| O7 | PyInstaller `.spec` template faltando | ⚠️ Menor | Incluir exemplo em `grupo14d-telemetry/examples/` |

---

## 7. Pontos fortes do SDD

- ✅ **Contexto claro**: Baseado em exploração real (P2-V2), não em suposição.
- ✅ **Especificação precisa**: Contrato v1 é bem definido, compatível com código existente.
- ✅ **Arquitetura modular**: `client.py`, `heartbeat.py`, `config.py` separados — bom design.
- ✅ **ADRs explícitas**: Justificativa clara para cada decisão (retry, heartbeat, backcompat).
- ✅ **Risco/mitigação**: Tabela de riscos está bem pensada (R1-R7).
- ✅ **Ordem de migração fundamentada**: Piloto primeiro, escalando. Critério sensato (baixo risco → confiança → críticos).
- ✅ **Sprints realistas**: A-E é 3-4 dias factíveis; F é longo mas esperado.

---

## 8. Pontos fracos do SDD

- ❌ **Backfill de nome**: Subespecificado, operacionalmente arriscado.
- ❌ **IRPF no final**: Contradição com "é a fonte do port e tem heartbeat"; deveria vir cedo pra validar.
- ❌ **Sem critério de "pronto pra próximo"**: Observação 3 dias é subjetiva; falta SLA.
- ❌ **Heartbeat 60s**: Custo no Neon não foi validado em baseline.
- ⚠️ **Config loader**: Ambiguidade em env vs config.ini vs hardcoded.
- ⚠️ **Sem health check público**: Difícil pra RPA diagnost  icar telemetria offline em produção.

---

## 9. Aprovação

**Voto: APROVADO COM CONDIÇÕES**

Antes de disparar código (Sprint A), resolver as 3 críticas bloqueantes:

1. ✏️ Reescrever `scripts/backfill-v1-naming.sql` conforme seção 1.1.
2. ✏️ Reordenar Sprint F: IRPF posição 2-3, adicionar SLA de migração.
3. ✏️ Integrar checklist de seção 2.1 antes de liberar próximo RPA.

Depois disso, verde para começar Sprint A.

---

## 10. Documento final para início

**Anexos a preparar (pré Sprint A):**

```
grupo14d-telemetry/
├── SDD_GRUPO14D_TELEMETRY_V0.1.md        (versão revisada, com críticas resolvidas)
├── RUNBOOK_SPRINT_A.md                   (instruções passo a passo pra scaffolding)
├── RUNBOOK_SPRINT_F_MIGRAÇÃO.md          (procedimento por RPA + SLA checklist)
└── scripts/
    └── backfill-v1-naming.sql            (versão robusta, com validation + backup)
```

Davi: revisa esta avaliação, sinaliza qualquer discordância, e eu reescrevo os documentos.

---

## 11. Próximos passos imediatos (após aprovação)

1. **Você:** Lê P3-AVALIACAO, marca como LIDO.
2. **Você:** Responde às 3 críticas: concorda com backfill robusto? Com IRPF mais cedo? Com SLA?
3. **Eu:** Reescrevo SDD com respostas, crio RUNBOOK_SPRINT_A.md.
4. **Você:** Cria repo `grupo14d-telemetry` vazio no GitHub.
5. **Eu:** Disparo prompt do Sprint A no Claude Code (scaffolding).

---

