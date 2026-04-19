# Changelog

## [Unreleased]

(vazio — nenhuma mudança desde o release)

## [0.1.0] - 2026-04-18

### Added

- **Hub de Governança 14D**: Portal centralizado de documentação com design unificado (Dark Mode, tipografia Newsreader/IBM Plex).
- **Obsidian SSG (Custom)**: Motor de geração de site estático que converte notas do Vault para o portal web, preservando wikilinks e estilo corporativo.
- **Docs Server (Mini API)**: Servidor FastAPI para roteamento dinâmico e suporte a documentação estática local.
- **Automação de README**: Sistema híbrido (GitHub Actions + Python) para padronização de READMEs em 15 RPAs com injeção segura de governança.
- **Sprint A**: Scaffolding completo do pacote (pyproject.toml, estrutura src/, módulos
  placeholder, CI/CD ausente por design).
- **Sprint B**: Classe `Telemetry` com:
  - Contrato v1 (`contract_version="v1"` injetado via `_send_payload` central)
  - Validação kebab-case estrita de `rpa_name`
  - Política de erro distinguida (URLError DEBUG, HTTPError 401/403 WARNING)
  - Propriedades thread-safe `is_healthy` e `last_error`
  - Enums `Event` e `Status` em `events.py`
  - Helpers em `_internal.py` (`new_session_id`, `get_hostname`, `utc_now_iso`)
- **Sprint C**: `HeartbeatDaemon` com:
  - `interval_seconds: float` (mínimo 0.1s)
  - `event_name` parametrizável (default `"heartbeat"`, aceita `"watcher_heartbeat"` pra compat IRPF)
  - `detalhes_provider` callable blindado contra exceções
  - `start()` / `stop()` idempotentes e thread-safe
  - Context manager (`__enter__` / `__exit__` não suprime exceção)
  - Thread nomeada `HeartbeatDaemon-<rpa_name>` pra debug
- **Sprint D**: Config loader multi-formato:
  - `TelemetryConfig` `@dataclass(frozen=True, slots=True)` — imutável
  - `ConfigNotFoundError` em auto-descoberta vazia
  - `load_config()` cascata 3 camadas (args > env > arquivo)
  - Env vars `G14D_TELEMETRY_*` primárias + `RPA_*` como fallback compat
  - Suporte INI (`config.ini`, `config-global.ini`) e YAML (`pipeline.yml`)
  - Resolução PyInstaller: `sys.executable` antes de `sys._MEIPASS`
  - Validação kebab-case com bypass controlado (`G14D_TELEMETRY_ALLOW_INVALID_NAME=1`)
- **Sprint E**: `api_key` ocultada em `repr()` do `TelemetryConfig` (preventivo).

### Dependencies

- Runtime: apenas stdlib (`urllib`, `threading`, `dataclasses`, `configparser`, `pathlib`)
- Opcional `[test]`: `pytest>=8.0`
- Opcional `[examples]`: `psutil>=5.9`
- Opcional `[yaml]`: `pyyaml>=6.0`

### Testing

- **77 testes** passando (Sprint B: 24, Sprint C: 22, Sprint D: 28, Sprint E: 3)
- Stress test de concorrência sem race conditions
- Stress test de start/stop concorrente sem deadlock

## [0.1.0.dev0] - 2026-04-17

- Release inicial de desenvolvimento (scaffolding apenas).
