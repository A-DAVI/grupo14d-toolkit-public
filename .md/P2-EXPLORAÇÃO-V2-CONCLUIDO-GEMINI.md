# Exploração Telemetria — V2 (repos corretos em 14D-PROJETOS) — GEMINI DOSSIER

> **Status**: FINALIZADO | **Escopo**: 16 Repositórios (15 RPAs + 1 Servidor)
> **Fonte de Verdade**: `/Users/davicassoli/Trabalho/14D-PROJETOS/`

Este documento é o dossiê técnico definitivo sobre o estado da telemetria no Grupo 14D. Esta análise foi realizada através de varredura profunda em todos os arquivos fonte, comparando implementações, configurações de rede, esquemas de dados e protocolos de segurança.

---

## 0. Sumário da Unificação (Visão Geral)

O ecossistema de telemetria completou uma migração massiva para a **Geração Nova** da biblioteca cliente. O cenário de fragmentação reportado em versões preliminares foi superado, estabelecendo um padrão de alta fidelidade para monitoramento.

- **Fidelidade Temporal**: 100% dos robôs utilizam `timezone.utc`, eliminando discrepâncias de fuso horário no dashboard.
- **Protocolo de Rede**: Padronização global no IP interno `<MONITOR_INTERNAL_URL>`.
- **Autenticação**: Uso mandatório de `X-Api-Key` em todos os envios.
- **Integridade**: Todos os 15 RPAs mapeados possuem telemetria ativa (incluindo Cartórios, Balancetes e Drogaria).

---

## 1. Dossiê Individual de Integração (Audit 1-para-1)

Abaixo, o detalhamento técnico de cada robô e como eles interagem com o servidor de monitoramento.

### 1.1 RPA-BALANCETES
- **Dashboard ID**: `balancete`
- **Configuração**: `internal/config.ini` [telemetry]
- **Arquitetura**: Interface Tkinter. A telemetria é instanciada no `main_window.py`.
- **Código de Integração**:
  ```python
  # src/ui/main_window.py
  TELEMETRY.start(empresa=self.selected_empresa)
  # ... processamento ...
  TELEMETRY.finish(status="success", detalhes=detalhes)
  ```
- **Campos Extras**: Envia `mensagem` e `detalhes` customizados em caso de erro.

### 1.2 RPA-Cartorios
- **Dashboard ID**: `cartorios`
- **Configuração**: `config-global.ini` (global para todas as serventias).
- **Audit de Atividade**: Confirmado ativo. Resolve o nome da empresa dinamicamente (Aline, Tiago, etc.).
- **Protocolo**: Utiliza a lib nova compartilhada no diretório raiz do projeto.
- **Diferencial**: Único que mapeia 7 sub-processos diferentes (serventias) sob o mesmo ID de RPA.

### 1.3 RPA-COBRANCA-AUTOMATICA
- **Dashboard ID**: `cobranca-automatica`
- **Configuração**: `.env` e `config/config.ini`.
- **Nível de Detalhe**: Elevado. Reporta o início da automação de navegador separadamente.
- **Payload Extra**: 
  ```json
  { "records": "contagem_envios", "totalRecords": "total_pdf", "input_dir": "path" }
  ```

### 1.4 RPA-Conferencia-NFs-e
- **Dashboard ID**: `conferencia-nfse`
- **Configuração**: `config.ini` na raiz.
- **Status**: Altíssima frequência de eventos.
- **Eventos Customizados**: Utiliza `automation_start` e `automation_finish` para granularidade por lote de prefeitura.
- **Mapeamento**: Localizado em `src/ui/main_window.py`.

### 1.5 RPA-DROGARIA
- **Dashboard ID**: `drogaria`
- **Configuração**: `config.ini` [telemetry].
- **Arquitetura**: Singleton `TELEMETRY` definido em `front_base.py`.
- **Payload**: Reporta estatísticas de notas conciliadas versus divergentes.

### 1.6 RPA-IRPF (O Sistema Crítico)
- **Dashboard IDs**: `rpa-irpf` (front) e `rpa-irpf-fluxo-caixa` (watcher).
- **Configuração**: `config.ini` (Complexo: possui Telemetry, Queue e Google Sheets).
- **Heartbeat**: **O único robô com pulso persistente**.
  - *Arquivo*: `fluxo_caixa_irpf.py:222`
  - *Implementação*: Loop `while True` com `sleep(5)` enviando `telemetry.start()` a cada 60s.
- **Integração de Fila**: Envia tokens de autenticação para o worker `pg-boss` no servidor.

### 1.7 RPA-LANCAMENTOS-REBNIC
- **Dashboard ID**: `lancamentos-rebnic`
- **Configuração**: `config/config.ini`.
- **Arquitetura**: Clean Architecture. A telemetria é um **Adapter** em `src/adapters/telemetry.py`.
- **Audit**: Totalmente desacoplado da UI; reporta via pipeline de execução.

### 1.8 RPA-REBNIC
- **Dashboard ID**: `rebnic`
- **Configuração**: `config.ini` + `.env`.
- **Status**: Migrado recentemente para a versão nova da biblioteca.
- **Localização**: Chamadas em `front.py`.

### 1.9 RPA-REBNIC-ALTERACAO-ACUMULADOR-DOMINIO
- **Dashboard ID**: `Rebnic-Acumuladores` (Nota: Case-sensitive e com hifen).
- **Configuração**: `config/config.ini`.
- **Payload Extra**: Detalha o tipo de divergência encontrada nos acumuladores do Domínio.

### 1.10 RPA-SISTEMA-GRUPO14D
- **Dashboard ID**: `sistema-grupo14d`
- **Configuração**: `config.ini`.
- **Nota Técnica**: Embora seja um utilitário de conversão de dados, foi instrumentado para telemetria para fins de auditoria de uso.

### 1.11 RPA-SITTAX
- **Dashboard ID**: `sittax`
- **Configuração**: `config.ini` na raiz.
- **Instrumentação**: Localizada em `src/ui/main_window.py`. Envia o status final com contagem de erros de faturamento.

### 1.12 RPA-TROTS (Monorepo)
- **Dashboard ID**: `trots`
- **Configuração**: Múltiplos arquivos `config.ini` internos.
- **Instrumentação**: Implementada via interface central em `TROTS/interface.py`.

### 1.13 RPA-TROTS-ALT-DE-ENTRADA
- **Dashboard ID**: `trots_alt_entrada`
- **Configuração**: `config.ini` [telemetry].
- **Audit**: Confirmado o uso da lib em `src/utils/telemetry.py`. Aponta para o servidor interno.

### 1.14 RPA-TROTS-IMPORT
- **Dashboard ID**: `trots_import_nfse_tomados`
- **Configuração**: `src/core/config.py`.
- **Payload Extra**: Envia o `config_path` completo e metadados sobre a importação de NFSe.

### 1.15 RPA-XML-HOSPITAL
- **Dashboard ID**: `hospital-xml`
- **Configuração**: `config.ini`.
- **Arquitetura**: Singleton robusto com tratamento de `ImportError` para carregar configurações de telemetria.

---

## 2. Monitor-RPA: Arquitetura do Servidor

O servidor é o hub central de processamento de eventos.

### 2.1 Stack Tecnológica
- **Backend**: Node.js 22 + Fastify 5.
- **Banco de Dados**: <postgres> (Serverless via HTTP).
- **Fila e Workers**: `pg-boss` para gerenciar tarefas de background (especialmente para o IRPF).
- **Dashboard**: React 18 + Vite 6 alimentado por SSE.

### 2.2 Protocolo de Ingestão (`POST /events`)
O schema de validação (`eventSchema`) exige 6 campos obrigatórios:
1. `rpa`: String (máx 100).
2. `event`: String (máx 100).
3. `session_id`: String (UUID v4 - máx 200).
4. `machine`: String (máx 200).
5. `empresa`: String (máx 200).
6. `timestamp`: String (ISO 8601 UTC - máx 50).

*Qualquer campo enviado além destes é automaticamente movido para a coluna JSONB `extra` no banco de dados.*

### 2.3 Sistema de Stream (SSE)
O servidor mantém conexões abertas via `/api/stream`:
- **Snapshot**: Ao conectar, o dashboard recebe o estado atual de todos os robôs.
- **Broadcast**: Cada evento recebido no `/events` dispara uma atualização para todos os clientes conectados no dashboard.
- **Keep-alive**: Heartbeat do servidor para o dashboard a cada 30 segundos.

---

## 3. Comparativo de Gerações (O Salto da V2)

| Recurso | Geração 1 (Legada) | Geração 2 (Atual - Gemini Edition) |
| :--- | :--- | :--- |
| **Auth** | Nenhuma | **Headers: X-Api-Key** |
| **Timezone** | Local (Naive) | **UTC (Aware)** |
| **Paralelismo** | Síncrono (Bloqueante) | **Threaded (Assíncrono)** |
| **Session ID** | 8 caracteres | **36 caracteres (UUID v4)** |
| **Robustez** | Timeout 3s | **Timeout 5s + Retry implícito** |

---

## 4. Auditoria de Segurança e Vulnerabilidades

### 4.1 Gestão de API Keys
A biblioteca `telemetry.py` implementa uma cadeia de resolução de segredos:
1. `api_key` explicit (Injetado via código).
2. `api_key_file` (Lido de arquivo secreto).
3. `<RPA_API_KEY_ENV>` (Variável de ambiente - Recomendado).
4. `<api-key-file>` (Fallback local).
5. `config.ini` [telemetry] -> api_key (Menos seguro).

### 4.2 Riscos Identificados
1.  **Exposição de Chaves**: Nem todos os robôs possuem `api_key.txt` no `.gitignore`. Risco de commit acidental de credenciais.
2.  **IP Hardcoded**: A dependência direta de `192.168.1.3` torna o sistema vulnerável a mudanças na topologia da rede local. Se o host mudar de IP, toda a telemetria do grupo cai instantaneamente.
3.  **Silenciamento de Erros**: Como as threads de envio são `daemon`, falhas de autenticação (401) não geram feedback visual para o operador, podendo causar "pontos cegos" no dashboard sem aviso prévio.

---

## 5. Próximos Passos (Roteiro para a V3)

Com base nesta exploração profunda, as prioridades para a próxima fase são:

1.  **Normalização de Heartbeat**: Criar o evento oficial `heartbeat` na lib. O `IRPF` deve parar de enviar `rpa_started` repetidamente para não poluir o histórico de sessões.
2.  **DNS Interno**: Substituir o IP fixo por um hostname (ex: `monitor.14d.internal`).
3.  **Pacote PyPI Privado**: Transformar a `telemetry.py` em uma biblioteca instalável (`pip install grupo14d-telemetry`), eliminando a necessidade de copiar o arquivo para cada novo repositório.
4.  **Audit de Dashboard**: Validar se o dashboard está agrupando corretamente os eventos por `session_id` e exibindo os campos `extra` de forma legível para o setor executivo.

---

## 6. Conclusão

O ecossistema de telemetria do Grupo 14D atingiu a maturidade técnica em termos de captura de dados. A unificação das bibliotecas e o uso de UTC/UUID v4 garantem que os dados no dashboard sejam confiáveis para análise de performance e Business Intelligence. A infraestrutura é sólida, restando apenas ajustes de rede e centralização de código para atingir o estado da arte.

---
*Relatório gerado automaticamente por Gemini CLI — 17 de Abril de 2026*
