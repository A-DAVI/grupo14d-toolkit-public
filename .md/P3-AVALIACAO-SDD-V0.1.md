# Avaliação Técnica — SDD `grupo14d-telemetry` v0.1

**Documento de Revisão Arquitetural (P3)**
**Avaliador:** Engenharia e Arquitetura
**Data:** 17 de Abril de 2026
**Documento Alvo:** `SDD_GRUPO14D_TELEMETRY_V0.1.md`
**Status do Parecer:** APROVADO (com ressalvas de infraestrutura)

---

## 1. Parecer Executivo

O **SDD v0.1** é um documento de especificação excepcionalmente maduro e pragmático. Ele ataca com precisão a dívida técnica primária do Grupo 14D (fragmentação de 15 cópias da biblioteca de telemetria) sem cair na armadilha do *overengineering*. 

A decisão de limitar o escopo estritamente à telemetria (excluindo parsers e regras de negócio) e de manter a retrocompatibilidade no servidor `MONITOR-RPA` demonstra excelente visão de controle de danos (Blast Radius). O plano de execução dividido em 6 Sprints incrementais torna a entrega tangível e segura.

---

## 2. Análise Arquitetural e Design de API

### 2.1 O que está excelente (Best Practices)
*   **Design de API Pública:** A redução da superfície de contato (exportando apenas 5 símbolos) protege o pacote de vazamento de contexto interno.
*   **Context Manager para Heartbeat:** O uso de `with HeartbeatDaemon(...)` é a abordagem mais idiomática em Python para gerenciar ciclos de vida de threads em background de forma segura, garantindo que o pulso comece e pare junto com o escopo do trabalho.
*   **Cascata de Resolução de Configuração:** A lógica `load_config` (Sessão 4.5) que herda as 6 formas diferentes de configuração que os RPAs usam hoje (`.ini`, `.yml`, `.env`, `.txt`) é a chave para o sucesso do *Drop-in Replacement*. Reduzirá a refatoração nos clientes a praticamente zero.

### 2.2 O que precisa de ajuste (Nível de Código)
*   **Validação de Naming (Kebab-case):** A regex sugerida `^[a-z0-9][a-z0-9-]*$` valida com sucesso `trots-alt-entrada`, mas também valida nomes sem hífen como `cartorios` e `balancete`. Isso é positivo (não quebra compatibilidade), mas deve ser coberto explicitamente nos testes unitários do Sprint B para garantir que não haja falsos positivos.

---

## 3. Vulnerabilidades e Pontos Cegos Identificados (Risco Técnico)

Embora o design de software esteja sólido, o SDD apresenta lacunas na camada de **Sistemas Operacionais e Redes** que podem travar os Sprints E e F.

| # | Risco Identificado | Nível | Impacto e Mitigação Sugerida |
|---|--------------------|-------|------------------------------|
| **V1** | **Build Quebrado via SSH** | **ALTO** | Na Sessão 6.3, a instalação sugere `pip install @ git+ssh://`. A maioria dos RPAs usa `PyInstaller` (compilação `.spec` ou `.bat`) em máquinas Windows. Máquinas de CI/CD ou desktops de operadoras não terão a chave SSH do GitHub. **Mitigação:** Alterar para `git+https://` utilizando um Personal Access Token (PAT) injetado no ambiente, ou empacotar via sub-módulos do git. |
| **V2** | **Falhas de Auth Silenciosas** | **MÉDIO** | A Sessão 4.6 define que falhas de rede dão *swallow* com log `DEBUG`. O servidor Fastify retorna HTTP `401` se a `X-Api-Key` for revogada/inválida. Esconder um 401 no nível DEBUG fará o robô rodar "cego" sem que a operação saiba. **Mitigação:** Distinguir `URLError` (rede fora) de `HTTPError` (401/403). Erros de autenticação devem lançar `logger.error` ou `logger.warning`. |
| **V3** | **Orfandade de Threads (Tkinter)** | **BAIXO** | O `HeartbeatDaemon` usa uma thread, e o `telemetry.send()` dispara outra. Em interfaces Tkinter (Cobrança, Cartórios, etc.), se as threads não forem marcadas como `daemon=True` e sofrerem um `.join()` rápido, o processo Python ficará travado em background após o fechamento da janela. **Mitigação:** Garantir que o `HeartbeatDaemon` injete `daemon=True` em suas workers. |

---

## 4. Avaliação das Decisões de Arquitetura (ADRs)

*   **ADR-01 (Pacote Separado):** `APROVADO`. Criar um monorepo (`toolkit`) agora forçaria dependências pesadas cruzadas. Um pacote micro (`grupo14d-telemetry`) sem dependências externas (usando `urllib` da *stdlib*) é a decisão arquitetural correta.
*   **ADR-02 (Sem fila persistente):** `APROVADO`. Em V0.1, simplicidade ganha da confiabilidade extrema. Se a rede `192.168.1.3` estiver estável, a perda de pacotes será residual.
*   **ADR-04 (Backfill SQL):** `APROVADO`. Manter a continuidade visual no dashboard e em queries de BI é crucial para a aderência da ferramenta pelo corpo executivo.
*   **ADR-05 (Server permissivo, Cliente restrito):** `APROVADO`. É o padrão de ouro em APIs distribuídas (Postel's Law: *Be conservative in what you send, be liberal in what you accept*). 

---

## 5. Análise do Plano de Rollout (Ordem de Migração)

A matriz de migração (Sessão 6.2) é um excelente modelo de contenção de risco (*Blast Radius Mitigation*).
*   Começar pelo `SISTEMA-GRUPO14D` (Utilitário interno, não crítico) e terminar no `RPA-IRPF` (Mais complexo, crítico para a operação atual) minimiza o impacto no negócio durante a curva de aprendizado da nova biblioteca.

---

## 6. Veredito e Próximos Passos

**Conclusão:** O documento SDD está maduro o suficiente para ser convertido em código. A modelagem de dados (Contrato V1) está perfeitamente alinhada com as necessidades da stack Fastify/Neon.

**Ações Imediatas Recomendadas:**
1. Atualizar o SDD original (Sessão 6.3) para prever o uso de `git+https` em vez de `git+ssh`.
2. Atualizar a política de log no SDD para exibir erros `401/403` como Warnings.
3. Criar o repositório `grupo14d-telemetry` no GitHub.
4. Avançar para a redação do **PROMPT do Sprint A**.
