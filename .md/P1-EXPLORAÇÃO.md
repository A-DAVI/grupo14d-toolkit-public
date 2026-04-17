# grupo14d-toolkit — Prompt de Exploração (pré-Sprint 1)

> **Para o Claude Code:** NÃO escreva código nesta fase. NÃO crie arquivos. NÃO instale nada. Sua missão é **investigar e reportar**. O output desta sessão é um relatório em markdown, nada mais.

---

## 0. Contexto

Vou construir uma biblioteca Python interna chamada `grupo14d-toolkit` pra centralizar padrões que se repetem em todos os RPAs do Grupo 14D. O primeiro módulo é **telemetria + heartbeat**, que conversa com o MONITOR-RPA.

Antes de começar a construir (Sprint 1 em diante, em outro documento), preciso que você **mapeie o terreno atual** — como a telemetria funciona hoje, disperso em cada RPA, e como o MONITOR-RPA recebe os eventos. Sem esse mapa, a biblioteca vai nascer desalinhada.

---

## 1. Fontes autorizadas de leitura

Leia **apenas** desses lugares. Não saia explorando o disco inteiro.

1. **Vault Obsidian:** `~/davicassoli/trabalho/grupo14d-obsidian-vault` (confirme o caminho exato no sistema — pode ter variação de nome). Procure por qualquer nota mencionando:
   - `telemetry`, `telemetria`, `heartbeat`
   - `MONITOR-RPA`, `monitor-rpa`, `monitor`
   - `agent skill` (nodes azuis)
   - Padrões/convenções dos RPAs
2. **Repositório do MONITOR-RPA** (backend Node.js/Fastify). Ache no disco e mapeie:
   - Endpoints que recebem eventos (rotas HTTP/SSE)
   - Schema do payload esperado
   - Tabelas no Postgres (Neon) relacionadas a eventos/telemetria
3. **Repositórios dos RPAs existentes.** Pelo menos estes, se encontrar:
   - `RPA-IRPF-016` (watcher)
   - `RPA-IRPF-017` (Google Sheets sync)
   - `RPA-REBNIC`
   - `RPA-Hospital` (XML)
   - `RPA-Conferencia-NFS-e`
   - `RPA-FLUXO-CAIXA`
   - Qualquer outro que apareça sob o mesmo diretório pai

**Se não achar algum, anota no relatório e segue.** Não invente.

---

## 2. Perguntas que o relatório precisa responder

### 2.1 Lado MONITOR-RPA (servidor)

- Qual URL base e porta?
- Quais endpoints recebem eventos? Liste cada um com método, path e exemplo de payload.
- Existe autenticação? Token, header, nada?
- Qual o formato do `timestamp` esperado? (ISO8601 UTC? epoch? string local?)
- Campos obrigatórios vs opcionais em cada evento
- O servidor dá resposta síncrona (200/4xx) ou aceita e processa async?
- Existe endpoint de heartbeat separado ou é o mesmo de log?
- SSE é só outbound (monitor → dashboard) ou os RPAs também usam?

### 2.2 Lado RPAs (clientes atuais)

Pra cada RPA que conseguir abrir:

- Como ele reporta hoje? (HTTP direto? biblioteca compartilhada via copy-paste? logging em arquivo?)
- Qual estrutura de payload ele manda? **Copie um exemplo real do código.**
- Manda heartbeat? Com que frequência? Qual coleta de métrica (CPU/mem)?
- Como trata falha de rede? (Crasha? Silencia? Retry?)
- Tem `run_id` / id de execução? Como gera?
- Usa `psutil`? Outra lib? Nada?

### 2.3 Divergências

A parte mais importante do relatório.

- Quais RPAs mandam eventos **diferentes** pra mesma coisa? (Ex: um chama `status: "ok"`, outro `status: "success"`)
- Quais campos aparecem em uns e faltam em outros?
- Alguém manda `rpa_name` inconsistente? (Ex: `RPA-IRPF-016` vs `irpf_016` vs `IRPF 016`)
- Alguma coisa quebrada ou comentada que indica que "a gente sabia que tinha que arrumar"?

### 2.4 Contexto adicional do vault

- Existe nota descrevendo o "jeito certo" de fazer telemetria? Ou só foi evoluindo orgânico?
- Tem decisão arquitetural registrada (ADR, nota de design)?
- Tem algum `TODO`, `FIXME`, ou nota "dívida técnica" mencionando telemetria?

---

## 3. Formato do relatório

Gere **um único arquivo markdown** em `/tmp/exploracao-telemetry.md` com esta estrutura:

```markdown
# Exploração Telemetria — Estado Atual

## Resumo executivo

(3-5 linhas: como está hoje, principal dor, recomendação preliminar)

## 1. MONITOR-RPA (servidor)

### 1.1 Endpoints

### 1.2 Schema de eventos

### 1.3 Autenticação

### 1.4 Observações

## 2. RPAs (clientes)

### 2.1 Inventário

| RPA | Caminho | Reporta? | Heartbeat? | Lib usada |
| --- | ------- | -------- | ---------- | --------- |

### 2.2 Detalhamento por RPA

#### RPA-IRPF-016

- ...
  (repetir por RPA)

## 3. Divergências e inconsistências

(lista objetiva)

## 4. Trechos de código relevantes

(snippets reais copiados, com caminho do arquivo e linha)

## 5. Contexto do vault

(links/notas relevantes encontradas)

## 6. Recomendação de contrato v1

(proposta inicial de schema unificado, baseada no que a maioria já faz)

## 7. Riscos e armadilhas

(o que pode quebrar na migração futura)

## 8. Perguntas abertas pro Davi

(lista de decisões que só o humano pode tomar)
```

---

## 4. Regras de engajamento

- **Só leia. Nunca escreva, nunca edite, nunca execute.** Exceção: o arquivo final do relatório.
- **Não invente.** Se não souber, escreva "não identificado" e segue.
- **Não dê opinião disfarçada de fato.** Opinião vai na seção 6 e 8, clara.
- **Copie código real, não paráfrase.** Em snippets, inclua caminho + linha.
- **Não rode nenhum RPA nem servidor.** Análise estática de código e leitura de notas apenas.
- **Timebox: 60-90 min.** Se passar disso, entrega o que tem e marca o que faltou na seção 8.
- **Pra cada afirmação importante, cite o arquivo:** `(fonte: /path/to/file.py:42)` ou `(fonte: vault/nota-x.md)`.

---

## 5. Critério de pronto

- [ ] Arquivo `/tmp/exploracao-telemetry.md` existe e segue o template da seção 3
- [ ] Todos os RPAs da lista em 1.3 aparecem (mesmo que como "não encontrado")
- [ ] Seção 3 (divergências) tem pelo menos 3 itens ou declara explicitamente "nenhuma divergência encontrada"
- [ ] Seção 6 (contrato v1) tem proposta concreta de JSON, não genérica
- [ ] Seção 8 (perguntas abertas) tem pelo menos uma pergunta real
- [ ] Nenhum arquivo fora de `/tmp/exploracao-telemetry.md` foi criado ou modificado

---

## 6. O que fazer depois

Pare. Me mostre o relatório. **Não siga pro Sprint 1 do `PLANO_TOOLKIT_V0.1.md`** até eu revisar e aprovar o contrato proposto. Se eu pedir ajustes no contrato, você atualiza o relatório e eu reviso de novo.

Começa.
