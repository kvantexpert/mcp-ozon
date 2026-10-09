# Ozon MCP — каноническое состояние проекта / точка продолжения

Дата актуализации: 2026-10-09

Перед любой следующей работой сначала читать этот файл и docs/KNOWLEDGE_BASE.md.

## 1. Repository

GitHub:
https://github.com/kvantexpert/mcp-ozon

Branch:
main

Примечание:
- фактическое состояние runtime на VPS проверяется отдельно от состояния GitHub-репозитория;
- текущий VPS runtime прошёл свежий read-only audit 2026-10-09.

## 2. Architecture

Seller and Performance are separate services on one VPS.

Seller:
- 127.0.0.1:8000
- public https://ozon-mcp.kvantexpert.ru/mcp
- MCP server runtime reports version 1.30.0

Performance:
- 127.0.0.1:8001
- no public route
- MCP server runtime reports version 1.30.0

Do not merge these runtimes.

## 3. Fresh VPS audit — 2026-10-09

Проверено без изменения runtime:

Services:
- ozon-mcp.service = active
- ozon-performance.service = active

Listeners:
- 127.0.0.1:8000
- 127.0.0.1:8001

Nginx:
- master process runs as root;
- worker process runs as www-data.
- локальный запуск `nginx -t` от desktop-agent не является валидной проверкой конфигурации: процесс не имеет доступа к TLS-сертификату. Конфигурацию/сертификаты не изменяли.

Public MCP:
- GET https://ozon-mcp.kvantexpert.ru/mcp -> HTTP 406; это ожидаемый ответ для Streamable HTTP при обычном GET.
- MCP initialize через public HTTPS -> HTTP 200;
- protocolVersion = 2025-03-26;
- serverInfo = ozon_mcp 1.30.0;
- MCP session успешно создана.

Ozon API READ:
- ozon_check_auth -> ready=true, source=env, missing_fields=[];
- ozon_get_products(visibility=ALL, limit=1) -> HTTP 200;
- total=170;
- получен product_id=6533359912, offer_id=4601546116680, sku=5964025559.

Вывод:
**Fresh VPS audit PASS. Public Seller MCP E2E READ PASS.**

## 4. Seller — доказано

Эталонная digital card:
- offer_id 4601546116680
- product_id 6417979753
- sku 5865629857
- category 200001489
- type_id 971075562
- price 23000 RUB
- VAT 0

Проверено:
- create;
- read;
- characteristics write/read;
- dictionary lookup;
- image import + primary;
- moderation approved;
- validation success;
- digital stock 0 -> 1;
- independent stock READ.

Не доказано:
- real order;
- real posting;
- digital code upload for real posting;
- delivery.

## 5. Catalog import — этап закрыт

Историческая задача массового импорта 293 позиций **не является текущей задачей**.

Фактический текущий статус:
- 169 позиций уже импортированы;
- этап импорта 169 закрыт;
- повторно импортировать их не требуется;
- к задаче 293 не возвращаться без отдельного явного запроса.

Старые записи о статусе `BLOCKED` для 293 являются историческими и не должны использоваться как текущий план работ.

## 6. Performance — текущий статус

Ранее выполнено:
- pinned 0.6.1 runtime;
- tracked 48-op catalog;
- 3 additions;
- safety corrections;
- systemd deployment;
- loopback :8001;
- 48/48 live describe audit.

Свежая проверка 2026-10-09:
- MCP initialize на 127.0.0.1:8001 прошёл;
- HTTP 200;
- protocolVersion = 2025-03-26;
- serverInfo = ozon_perf_mcp 1.30.0.

### Performance READ coverage — 2026-10-09

Проверен реальный E2E READ через runtime `ozon_perf_call_method`:

1. `ozonperf_get_api_client_campaign`
   - GET `/api/client/campaign`
   - safety = read
   - profile = `PERFORMANCE-RU`
   - HTTP 200
   - data: `list=[]`, `total="0"`

2. `ozonperf_get_api_client_statistics_campaign_product`
   - GET `/api/client/statistics/campaign/product`
   - safety = read
   - profile = `PERFORMANCE-RU`
   - HTTP 200
   - API вернул корректную CSV-структуру статистики; фактических кампаний нет, поэтому возвращена строка заголовков.

3. `ozonperf_get_api_client_campaign`
   - GET `/api/client/campaign`
   - safety = read
   - profile = `PERFORMANCE-BY`
   - HTTP 200
   - data: `list=[]`, `total="0"`

Дополнительно подтверждено:
- `ozon_perf_call_method` действительно является READ-only runtime tool;
- без profile вызов блокируется с требованием `PERFORMANCE-RU` или `PERFORMANCE-BY`;
- `PERFORMANCE-RU` и `PERFORMANCE-BY` оба реально проходят E2E READ;
- write/delete операции в ходе coverage не выполнялись;
- 48/48 catalog visibility + реальные READ ответы теперь вместе подтверждают не только загрузку каталога, но и фактическую работу нескольких безопасных Performance endpoints.

## 7. Minimal Seller AI-visible READ set — 2026-10-09

Зафиксирован минимальный подтверждённый Seller READ-набор:

1. `ozon_check_auth` — typed auth/read helper.
2. `ozon_product_list` — список товаров.
3. `ozon_product_info_list` — детали товаров.
4. `ozon_product_attributes` — характеристики товаров.
5. `ozon_category_tree` — актуальное дерево категорий/type_id.
6. `ozon_category_attributes` — атрибуты категории/type_id.
7. `ozon_stocks_info` — остатки и резерв.
8. `ozon_prices_get` — цены/комиссии/индексы цен.

`ozon_search_attributes_dictionary` намеренно исключён:
- operation помечена runtime как `UNVERIFIED`;
- точный контракт `SearchAttributeValuesRequest` не подтверждён;
- upstream `endpoints.yaml` не содержит body/params;
- успешный business READ не получен.

WRITE/DELETE операции в минимальный AI-visible набор не входят.

## 8. Current next steps

1. Fresh VPS audit — **DONE / PASS**.
2. Public Seller MCP initialize — **DONE / PASS**.
3. Public Seller Ozon READ `ozon_check_auth` — **DONE / PASS**.
4. Public Seller Ozon READ `ozon_get_products(limit=1)` — **DONE / PASS**.
5. Performance READ coverage — **DONE / PASS**.
6. Minimal Seller AI-visible READ set — **DONE / 8 operations confirmed**.
7. Следующий этап — применить этот набор к AI-visible поверхности без изменения WRITE/DELETE runtime.
8. Digital posting/code/delivery remains a separate later stage.

Не делать:
- не импортировать повторно 169 позиций;
- не возвращаться к массовому импорту 293 без явного запроса;
- не трогать `ozon-performance.service` при работах, не связанных с Performance;
- не менять systemd/nginx/runtime без отдельного основания и проверки.

## 9. Canonical documents

- docs/PROJECT_STATE_2026-09-27.md
- docs/KNOWLEDGE_BASE.md
- docs/SETUP_AND_RECOVERY.md
- server/SERVER_AUDIT_2026-09-27.md
- server/SERVER_STATE.md
- server/VPS_RUNTIME.md
- server/PERFORMANCE_RUNTIME.md
- docs/OZON_OPERATIONS.md
- docs/MASS_CATALOG_IMPORT_PLAN.md
- docs/PERFORMANCE_API_MATRIX_2026-09-26.md
- docs/PERFORMANCE_MCP_BASELINE_2026-09-26.md
- docs/TEST_HISTORY.md
- docs/AI_MAINTENANCE.md
