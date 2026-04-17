# grupo14d-toolkit

Hub de documentação, planejamento e especificação para o ecossistema de telemetria do Grupo 14D.

---

## O que é

Este repositório centraliza:

- **Exploração técnica** do estado atual da telemetria (arquivos P1, P2, P3)
- **Especificação** do contrato v1 de telemetria (SDD)
- **Plano de execução** para consolidação da lib em pacote compartilhado
- **Referências** ao vault de contexto do projeto

Não contém código de produção — é documentação e planejamento.

---

## Estrutura

```
grupo14d-toolkit/
├── .md/
│   ├── README.md                          (este arquivo)
│   ├── P1-EXPLORAÇÃO.md                   (template vazio)
│   ├── P1-EXPLORAÇAO-CONCLUIDO.md         (exploração inicial, 428 linhas)
│   ├── P2-EXPLORAÇÃO-V2.md                (template vazio)
│   ├── P2-EXPLORAÇÃO-V2-CONCLUIDO.md      (re-exploração, repos atualizados, 325 linhas)
│   ├── P2-EXPLORAÇÃO-V2-CONCLUIDO-GEMINI.md (dossiê narrativo, perspectiva alternativa)
│   └── P3-AVALIACAO-SDD-V0.1-CLAUDE.md    (avaliação crítica do SDD, 625 linhas)
├── SDD_GRUPO14D_TELEMETRY_V0.1.md         (especificação v0.1, 477 linhas)
└── README.md                              (este arquivo)
```

---

## Documentos principais

### P1 — Exploração Inicial (2026-04-17)

**Arquivo:** `.md/P1-EXPLORAÇAO-CONCLUIDO.md`

Estado da telemetria baseado em clones desatualizados. Contém:
- Mapeamento do MONITOR-RPA (servidor)
- Inventário dos 6 RPAs mapeados à época
- Divergências e inconsistências
- Template de contrato v1 preliminar
- Perguntas abertas para validação

**Status:** Obsoleto (baseado em repos antigos). Mantido como referência histórica.

---

### P2 — Re-exploração em 14D-PROJETOS (2026-04-17)

**Arquivo:** `.md/P2-EXPLORAÇÃO-V2-CONCLUIDO.md`

Re-exploração com repositórios atualizados de `/Users/davicassoli/Trabalho/14D-PROJETOS/`. Mapeamento completo:
- 15 RPAs confirmados, todos reportando telemetria
- Uma única geração de lib (`telemetry.py`, idêntica em todos)
- Mudanças no MONITOR-RPA (refactor `apps/api` + `apps/web`, novos endpoints)
- Heartbeat real em IRPF (300s, evento `watcher_heartbeat`)
- URL única `<MONITOR_INTERNAL_URL>` em produção
- Divergências operacionais (nomenclatura, fragmentação, falta de heartbeat em 14 RPAs)

**Status:** Atual. Fonte de verdade para estado real da telemetria.

---

### P3 — Avaliação Crítica do SDD (2026-04-17)

**Arquivo:** `.md/P3-AVALIACAO-SDD-V0.1-CLAUDE.md`

Avaliação rigorosa do SDD v0.1 de `grupo14d-telemetry`. Contém:
- 3 críticas bloqueantes (backfill SQL, ordem de IRPF, SLA de migração)
- 7 observações importantes (config loader, logging, heartbeat, testes, etc)
- Validação contra estado real (P2)
- Factibilidade de sprints A-E
- Cronograma revisto
- Aprovação com condições

**Status:** Pronto para revisão por Davi. Bloqueador das 3 críticas antes de começar Sprint A.

---

### SDD — Especificação v0.1 (2026-04-17)

**Arquivo:** `SDD_GRUPO14D_TELEMETRY_V0.1.md`

Especificação-Driven Development para `grupo14d-telemetry` v0.1. Contém:
- Contrato v1 formalizado (payload, eventos canônicos, autenticação)
- API pública do pacote (`Telemetry`, `HeartbeatDaemon`, `load_config`)
- 5 sprints de execução (A-E: scaffolding, port, heartbeat, config, release)
- Sprint F: ordem de migração dos 15 RPAs
- Riscos e mitigações (R1-R7)
- ADRs embutidas (decisões justificadas)
- Perguntas abertas (Q1-Q6, não-bloqueantes para v0.1)
- Critério de sucesso global

**Status:** Em revisão. Aguardando feedback sobre 3 críticas de P3 antes de início.

---

## Cronograma (proposto)

```
Sprint A (scaffolding):         meio dia          2026-04-18
Sprint B (port + contract):     1.5 dias          2026-04-18 PM — 2026-04-19
Sprint C (heartbeat):           1 dia             2026-04-20
Sprint D (config loader):       meio dia          2026-04-20 PM
Sprint E (release + backfill):  0.5 dias + 2h obs 2026-04-21 PM — 2026-04-22 obs

Sprint F (migração 15 RPAs):    3-4 semanas       2026-04-22 — 2026-05-20

Total: ~3.5 semanas até 15/15 migrados
```

Veja seção 6 de P3 para cronograma detalhado e rationale.

---

## Próximos passos

1. **Você revisa:** P3-AVALIACAO, especialmente as 3 críticas bloqueantes.
2. **Você aprova ou discorda:** Das recomendações em P3 seção 6.
3. **Eu reescrevo:** SDD com ajustes, crio RUNBOOK_SPRINT_A.md.
4. **Você cria:** Repo `grupo14d-telemetry` vazio no GitHub (irmão de `grupo14d-toolkit`).
5. **Eu disparo:** Prompt do Sprint A no Claude Code (scaffolding).

---

## Referências

### Repositórios relacionados

- **14D-PROJETOS** (fonte de verdade): `/Users/davicassoli/Trabalho/14D-PROJETOS/`
  - MONITOR-RPA (servidor)
  - RPA-BALANCETES, RPA-Cartorios, ..., RPA-XML-HOSPITAL (clientes)

- **grupo14d-obsidian-vault** (contexto): `/Users/davicassoli/Trabalho/grupo14d-obsidian-vault/`
  - Contextos/CONTEXTO - MONITOR-RPA.md
  - Projetos/MONITOR-RPA/Home.md
  - Projetos/RPA - COBRANÇA AUTOMATICA/Modulos/Pronto/08 - Telemetria.md

- **grupo14d-telemetry** (futuro): Repo a ser criado (scaffolding Sprint A)

### Convenções

- Todos os arquivos de exploração/avaliação vivem em `.md/`
- SDD e documentação técnica vivem na raiz
- Template files (P1-EXPLORAÇÃO.md, P2-EXPLORAÇÃO-V2.md) ficam vazios; conteúdo vai em `-CONCLUIDO`
- Commits em português, seguindo padrão `docs(telemetry): …`, `feat: …`, `chore: …`
- SSH para push via `git@github.com:A-DAVI/grupo14d-toolkit.git`

---

## Status do repositório

- P1: Obsoleto (histórico)
- P2-V2: Atual (fonte de verdade)
- P3: Pronto para revisão
- SDD: Aguardando resolução de 3 críticas
- Código: Nenhum (documentação apenas)

Última atualização: 2026-04-17 11:10 UTC

---
