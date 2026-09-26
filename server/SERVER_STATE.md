# Состояние сервера Ozon MCP

> Каноническая точка продолжения: `docs/PROJECT_STATE_2026-09-27.md`. Аудит VPS: `server/SERVER_AUDIT_2026-09-27.md`.

Фактическая фиксация текущего VPS: **26 сентября 2026**.

## Сервер

- ОС: Ubuntu 22.04.4 LTS (Jammy Jellyfish)
- Host: `cv7976275`
- Seller MCP backend: `127.0.0.1:8000`
- Performance MCP backend: `127.0.0.1:8001`
- Nginx: 80/443
- Seller systemd service: `ozon-mcp.service`
- Performance systemd service: `ozon-performance.service`
- оба сервиса включены и работают
- оба сервиса работают от root
- Seller credentials: `/root/.config/ozon-mcp/env`
- Performance credentials: `/root/.config/ozon-mcp/perf.env`
- credentials: `600 root:root`
- секреты в GitHub не хранятся.

## Архитектура

```
Internet
   |
   +--> Nginx HTTPS
   |       |
   |       +--> Seller MCP :8000
   |              |
   |              +--> Ozon Seller API
   |
   +--> Performance MCP: НЕТ публичного route
           |
           +--> 127.0.0.1:8001
                  |
                  +--> OAuth client_credentials
                  |
                  +--> Ozon Performance API
```

Performance intentionally remains local-only. Это отдельный сервис и отдельный credential set.

## Seller MCP

Production Seller MCP сохранен без изменения поведения.

Запуск:

`/root/.local/bin/uvx --from ozon-mcp-ru ozon-mcp-ru`

Версия:

- `ozon-mcp-ru 0.6.0`

Backend:

`127.0.0.1:8000`

Публичный endpoint:

`https://ozon-mcp.kvantexpert.ru/mcp`

Seller MCP не входил в Performance migration и должен оставаться независимым.

## Performance MCP

Отдельный сервис:

`ozon-performance.service`

Pinned runtime:

- `marketplaces-mcp-ru 0.6.1`
- upstream commit `ec2114595695536e001e09e1144a357118852db1`

Фактический runtime:

`/opt/kvantexpert/marketplaces-mcp-ru/.venv/bin/ozon-perf-mcp`

Backend:

`127.0.0.1:8001`

Транспорт:

Streamable HTTP.

Публичного endpoint нет.

## Performance catalog

Источник tracked patch:

`patches/marketplaces-mcp-ru/perf_endpoints.yaml`

Количество:

**48**

Установленный runtime также подтвержден как 48-operation catalog.

Автоматические проверки:

`tests/test_performance_catalog_patch.py`

Live-аудит 26.09.2026:

**48/48 operation_id успешно возвращены через `ozon_perf_describe_method`; 0 FAIL.**

## Новые операции

Подтверждены в live runtime:

1. `POST /api/client/statistics/products/sku` — `read`
2. `PATCH /api/client/campaign/{campaignId}` — `write`
3. `GET /api/client/campaign/all_sku_promo/set_bid` — `write`

## Safety corrections

В runtime подтверждены:

- `POST /api/client/min/sku` → `read`
- `POST /api/client/search_promo/bids/recommendation` → `read`
- `GET /api/client/campaign/all_sku_promo/activate` → `write`
- `GET /api/client/campaign/all_sku_promo/deactivate` → `write`
- `GET /api/client/campaign/all_sku_promo/set_bid` → `write`

Delete операции остаются `destructive`.

## Credentials

Файл:

`/root/.config/ozon-mcp/perf.env`

Содержит только Performance credentials и runtime variables.

Реальные значения не фиксируются в GitHub.

## Проверки

В текущей контрольной точке подтверждены:

1. Performance OAuth;
2. Bearer token;
3. прямой Performance API request;
4. MCP initialize;
5. MCP tools/list;
6. MCP tools/call;
7. describe_method;
8. live catalog audit 48/48.

Ранее контрольный read `GET /api/client/campaign` возвращал HTTP 200 и пустой список кампаний.

Канонический следующий шаг Performance: post-migration harmless READ `ozon_perf_call_method(operation_id=ozonperf_get_api_client_campaign)`.

## Deployment structure

Repository:

`https://github.com/kvantexpert/ozon-mcp`

Ключевые файлы Performance:

- `patches/marketplaces-mcp-ru/perf_endpoints.yaml` — runtime catalog patch;
- `tests/test_performance_catalog_patch.py` — catalog regression tests;
- `deploy/scripts/install-performance-mcp.sh` — reproducible installer;
- `deploy/systemd/ozon-performance.service` — production systemd unit;
- `docs/PERFORMANCE_API_MATRIX_2026-09-26.md` — 48-operation API contract;
- `docs/PERFORMANCE_PATCH_PLAN_2026-09-26.md` — implementation record;
- `server/PERFORMANCE_RUNTIME.md` — VPS runtime state;
- `server/SERVER_STATE.md` — overall VPS state;
- `docs/PERFORMANCE_MCP_BASELINE_2026-09-26.md` — baseline and next steps.

## Recovery

### Seller

Ubuntu → uv → nginx/certbot → env → systemd → MCP :8000 → HTTPS → Ozon Seller API.

### Performance

Ubuntu → pinned upstream → patched catalog → local venv → `perf.env` → systemd → MCP :8001 → OAuth → Ozon Performance API.

## Не переносится

Не включаются:

- credentials;
- TLS private key;
- TLS certificates;
- журналы;
- uv cache;
- реальные цифровые коды;
- Ozon API secrets;
- Performance client secret.

## Следующая точка разработки

1. Post-migration read smoke-test.
2. Read API coverage.
3. Минимальный набор AI-visible Performance tools.
4. Отдельное тестирование write/confirmation.
5. Архитектура внешнего доступа.

До принятия отдельного решения Performance MCP остается loopback-only.
