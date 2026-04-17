# grupo14d-toolkit — Prompt de Exploração v2 (completo)

> **Para o Claude Code:** NÃO escreva código de produção. NÃO crie arquivos fora do relatório. Sua missão é **mapear tudo**, não uma fatia. O relatório final vai decidir o escopo do toolkit v0.1.

---

## 0. Contexto

A rodada v1 de exploração foi incompleta — mapeou apenas 6 RPAs visíveis localmente. A verdade está no servidor da empresa, agora clonada no Mac. Você precisa refazer a exploração **sobre o inventário completo**.

Relatório v1 existe em `.md/` — **LEIA PRIMEIRO**. Não repita o que ele já concluiu. Amplie, corrija, e diferencie.

---

## 1. Escopo (ampliado)

### Fontes autorizadas

1. **Diretório raiz dos RPAs clonados:** `~/Trabalho/14D-PROJETOS/` (ou o caminho que o Davi informar no início da sessão).
2. **Vault Obsidian:** `~/Trabalho/grupo14d-obsidian-vault/` (ou caminho similar — confirmar).
3. **Repositório MONITOR-RPA** — já mapeado no v1, só consultar se precisar de referência cruzada.

### Quais RPAs entram no mapa

**TODOS.** Inclusive:

- RPAs que reportam telemetria hoje
- RPAs que NÃO reportam, mas deveriam
- RPAs legados, deprecados, abandonados (marcar como tal)
- RPAs experimentais ou protótipos
- Projetos que têm "RPA" no nome mas talvez não sejam automação (marcar como "não-RPA")

O objetivo é enxergar o **universo completo**, não só o que tá vivo.

---

## 2. Missão

Produzir um relatório consolidado que responda:

### 2.1 Inventário completo

Tabela obrigatória, uma linha por projeto no diretório raiz:

| Nome | Tipo | Stack | Em produção? | Reporta telemetria? | Tem heartbeat? | Como identifica (`rpa=`) | Última atividade (git log) | Observação |
| ---- | ---- | ----- | ------------ | ------------------- | -------------- | ------------------------ | -------------------------- | ---------- |

`Tipo` ∈ {RPA automação, dashboard, servidor, biblioteca, deprecated, desconhecido}.

### 2.2 Taxonomia de RPAs

Depois da tabela, agrupe os RPAs em **famílias**:

- Fiscal (IRPF, NFS-e, Conferência...)
- Cartórios
- Contábil (REBNIC, Fluxo de Caixa...)
- Outros

Pra cada família, responda:

- Quantos RPAs?
- Padrão comum de código que se repete entre eles?
- Divergências internas da família?

### 2.3 Padrões de telemetria — mapa consolidado

Amplia a análise v1 com os novos RPAs descobertos:

**A.** Pra cada RPA que reporta hoje, qual padrão usa?

- `telemetry.py` copiado do Cartorios
- `telemetry.py` copiado do REBNIC
- Implementação própria (qual?)
- Nenhum

**B.** Quantas cópias da `telemetry.py` existem? Elas divergem? Em quê?

**C.** Pra cada RPA que NÃO reporta, qual seria a dificuldade de adicionar? Classifica em:

- **Trivial** — RPA tem GUI/entrypoint claro, basta plugar lib
- **Médio** — precisa refatorar ponto de entrada
- **Difícil** — arquitetura não comporta (ex: Google Apps Script, script one-shot)

### 2.4 Padrões ALÉM de telemetria

Essa seção é nova e crítica. Pro toolkit não ficar preso em só telemetria, identifique outros padrões que se repetem:

- **Parsers:** quantos RPAs parseiam xls/xlsx? XML? PDF? Quais libs usam?
- **Login em portais:** quantos fazem login? Em quais sistemas (Domínio, Prefeitura, eSocial, e-CAC)?
- **Conciliação:** quantos fazem matching de registros? Que chave usam?
- **Output:** quantos geram Excel? Google Sheets? PostgreSQL?
- **Packaging:** quantos usam PyInstaller? Com `.spec` customizado?
- **GUI:** quantos têm Tkinter/ttkbootstrap? Quantos são headless/daemon?
- **Config:** como cada RPA lê config? env var, `config.ini`, hardcoded?

Pra cada categoria, estime **% de RPAs que compartilhariam uma solução comum**.

### 2.5 Ranking de dor

Com base em tudo acima, ordene as dores candidatas pro toolkit v0.1:

1. \_\_\_ (impacto: X RPAs, esforço de extração: Y)
2. ***
3. ***
   ...

**Impacto** = quantos RPAs se beneficiam.
**Esforço de extração** = quanto código precisa ser extraído/unificado.

### 2.6 Resposta às perguntas abertas do v1

O relatório v1 terminou com 9 perguntas abertas (seção 8). Com o inventário completo agora, responda as que viraram respondíveis. Marque explicitamente as que seguem sem resposta.

---

## 3. Formato do relatório

Arquivo único em `/tmp/exploracao-telemetry-v2.md`:

```markdown
# Exploração v2 — Mapa Completo

## Resumo executivo

(5-8 linhas: total de RPAs, quantos reportam, principal achado novo, recomendação refinada de v0.1)

## 1. Inventário completo

(tabela da seção 2.1)

## 2. Famílias de RPAs

(seção 2.2)

## 3. Telemetria — mapa consolidado

### 3.1 Quem reporta e como

### 3.2 Cópias da lib e drift

### 3.3 Dificuldade de adicionar aos que não reportam

## 4. Outros padrões repetidos

(parsers, auth, conciliação, output, packaging, GUI, config)

## 5. Ranking de dor pro toolkit

(com impacto e esforço quantificados)

## 6. Respostas às perguntas abertas do v1

(cada uma das 9, respondida ou marcada "segue aberta")

## 7. Novas perguntas abertas pro Davi

(o que só ele pode decidir agora)

## 8. Recomendação final de escopo do v0.1

(proposta concreta baseada em evidência, não em opinião)
```

---

## 4. Regras de engajamento

- **Só leia. Nunca escreva, nunca execute.** Exceção: o próprio relatório.
- **Não invente.** Se um campo da tabela não for determinável, escreva `?`.
- **Cite fonte** em toda afirmação importante: `(fonte: repo-X/arquivo.py:linha)`.
- **Git log rápido:** pra "última atividade", usa `git -C <repo> log -1 --format=%cd`.
- **Não rode nada:** nem `pip install`, nem `python arquivo.py`. Análise estática.
- **Timebox: 90-120 min.** Se passar, entrega o que tem e marca lacunas.
- **Se achar mais de 20 RPAs**, avisa o Davi antes de continuar — pode ser que tenha repo não-RPA que casou o filtro.

---

## 5. Critério de pronto

- [ ] Arquivo `/tmp/exploracao-telemetry-v2.md` existe
- [ ] Tabela da seção 1 tem TODA linha do diretório `~/Trabalho/rpas-all/`
- [ ] Seção 4 (outros padrões) tem pelo menos 5 categorias preenchidas
- [ ] Seção 5 (ranking) tem números, não adjetivos
- [ ] Seção 6 responde às 9 perguntas do v1 uma a uma
- [ ] Seção 8 tem recomendação única e justificada
- [ ] Nenhum arquivo além do relatório foi criado ou modificado

---

## 6. O que fazer depois

**Pare.** Me mostre o relatório. Não vá pro Sprint 1. A decisão de escopo do v0.1 depende desse mapa.
