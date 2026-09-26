# Ozon MCP — каноническое состояние проекта / точка продолжения

Дата фиксации: 2026-09-27

Перед любой следующей работой сначала читать этот файл.

## 1. Репозиторий

- GitHub: https://github.com/kvantexpert/ozon-mcp
- branch: main
- HEAD: eaddd8f6cc42b3199817230f658ee016e40f58c0
- последний commit: docs: mark Performance API matrix live at 48 operations
- commit statuses: statuses=[]; обязательного CI gate нет

Репозиторий одновременно является deployment layer и долговременной базой знаний Seller MCP + Performance MCP.

## 2. Архитектура

Один VPS содержит два независимых сервиса.

Internet
  -> Nginx HTTPS
     -> Seller MCP 127.0.0.1:8000 -> Ozon Seller API
  -> Performance MCP 127.0.0.1:8001 -> OAuth -> Ozon Performance API

Seller public endpoint: https://ozon-mcp.kvantexpert.ru/mcp
Performance public endpoint: НЕТ. Порт 8001 должен оставаться loopback-only до отдельного access-control решения.

Seller и Performance не объединять в один runtime.

## 3. VPS

Последнее документированное состояние VPS: 2026-09-26.

- host: cv7976275
- OS: Ubuntu 22.04.4 LTS
- nginx: 80/443
- Seller backend: 127.0.0.1:8000
- Performance backend: 127.0.0.1:8001
- Seller service: ozon-mcp.service
- Performance service: ozon-performance.service
- оба сервиса в последней зафиксированной проверке enabled/active
- оба запускаются от root

Seller:
- package: ozon-mcp-ru 0.6.0
- фактический unit запускает: /root/.local/bin/uvx --from 'ozon-mcp-ru==0.6.0' ozon-mcp-ru
- env: /root/.config/ozon-mcp/env

Performance:
- package: marketplaces-mcp-ru 0.6.1
- pinned upstream: ec2114595695536e001e09e1144a357118852db1
- runtime: /opt/kvantexpert/marketplaces-mcp-ru/.venv/bin/ozon-perf-mcp
- env: /root/.config/ozon-mcp/perf.env
- permissions for env files: 600 root:root

## 4. Seller MCP — что доказано

Эталонная карточка:
- offer_id: 4601546116680
- product_id: 6417979753
- sku: 5865629857
- category: 200001489
- type_id: 971075562
- name: 1С:Бухгалтерия 8 ПРОФ. Электронная поставка
- price: 23000 RUB
- VAT: 0

Подтверждено:
- создание карточки;
- чтение карточки;
- изменение characteristics;
- dictionary lookup;
- изображение + primary image;
- moderation approved;
- validation success;
- digital stock 0 -> 1;
- независимый stock READ: present=1, reserved=0.

Не доказана полностью цепочка ORDER -> POSTING -> CODE -> DELIVERY.
Доказано: CARD -> STOCK.

## 5. Seller operations — важные факты

Product import: POST /v3/product/import.
- максимум 100 items за запрос;
- asynchronous task_id;
- status через POST /v1/product/import/info.

Для реального payload учитывать как минимум:
- offer_id;
- description_category_id;
- type_id;
- price;
- currency_code;
- depth, width, height, dimension_unit;
- weight, weight_unit;
- attributes;
- images при необходимости.

Сокращённое MCP wrapper description нельзя считать полной схемой Ozon Seller API.

Characteristics: POST /v1/product/attributes/update — WRITE через ozon_write_method с подтверждением.
Images: POST /v1/product/pictures/import — WRITE.
Digital stock: POST /v1/product/digital/stocks/import — WRITE.
Independent stock check: POST /v4/product/info/stocks.

Старый POST /v1/product/upload_digital_codes проверен и вернул 404 даже напрямую с VPS. Не использовать.
Для posting-based delivery исследована POST /v1/posting/digital/codes/upload; без реального posting_number не вызывать.

## 6. Массовый импорт 293

Источник: GOODS.JSON. Исходный файл не менять.

Точный набор:
- group 00000002 = 193;
- group 00000003 = 100;
- total = 293.

Исключены:
- 00000292 = 28;
- 00000296 = 4;
- 00000314 = 2.

Партии: 100 + 100 + 93.

Уже сделано:
- GOODS.JSON исследован;
- набор 293 определён;
- найден product import endpoint;
- найден import info endpoint;
- правило цены 00001 -> 00003 -> 00004 зафиксировано;
- Basic/PROF/CORP и прочие варианты не объединять автоматически.

### Блокер

Последний live snapshot нового аккаунта:
- total 0/500;
- daily_create 2/1500;
- daily_update 0/20000;
- rate 30000/min;
- category tree: 200001489 и 971075562 -> disabled=true;
- import без category -> description_category_is_empty;
- import с 200001489 -> used_forbidden_category.

Главный блокер — доступность category/type в новом аккаунте. Это не лимит 500.

До снятия блокера 293 не отправлять.

После снятия блокера:
1. свежий READ category tree;
2. проверить disabled=false для конечной category/type;
3. сформировать payload;
4. локально проверить payload;
5. WRITE 100;
6. получить task_id;
7. product/import/info;
8. зафиксировать результат;
9. повторить 100 и 93.

## 7. Performance MCP — завершённый этап

Цель: перейти с исходного 45-operation catalog на audited 48-operation catalog, не изменяя Seller MCP.

Runtime:
- marketplaces-mcp-ru 0.6.1;
- pinned upstream ec2114595695536e001e09e1144a357118852db1;
- tracked patch patches/marketplaces-mcp-ru/perf_endpoints.yaml;
- runtime catalog 48 operations;
- service ozon-performance.service;
- backend 127.0.0.1:8001.

Добавлены:
- POST /api/client/statistics/products/sku -> read;
- PATCH /api/client/campaign/{campaignId} -> write;
- GET /api/client/campaign/all_sku_promo/set_bid -> write.

Safety corrections:
- POST /api/client/min/sku -> read;
- POST /api/client/search_promo/bids/recommendation -> read;
- GET activate/deactivate all SKU promo -> write;
- GET set_bid -> write.

Live audit 26.09.2026:
- ozon_perf_describe_method по всем 48 operation_id;
- 48/48 found;
- 0 FAIL.

Это доказывает загрузку patched catalog и видимость operation_id в runtime. Это не доказывает успех каждого бизнес-вызова и особенно write-операций.

## 8. Performance — точка продолжения

Первый следующий smoke-test:
ozon_perf_call_method(operation_id=ozonperf_get_api_client_campaign)

Затем:
1. READ coverage;
2. определить минимальный AI-visible tool set;
3. отдельно тестировать WRITE с confirmation;
4. спроектировать auth/access-control;
5. только потом решать вопрос nginx/public exposure.

## 9. Принцип работы

DISCOVER -> READ -> VALIDATE -> WRITE -> VERIFY -> DOCUMENT

Для новой operation:
1. search/найти operation;
2. describe;
3. проверить method/path/safety/request schema;
4. сделать READ, если возможно;
5. WRITE только после проверки;
6. task/status;
7. независимый READ;
8. записать результат в docs.

## 10. Что нельзя повторять

- used_forbidden_category = проверить доступность category/type, не повторять массовый WRITE;
- description_category_is_empty = отсутствует обязательная category;
- 404 /v1/product/upload_digital_codes = stale endpoint;
- 400 deprecated /v1/posting/digital/list = сначала искать актуальную v2 operation;
- dictionary IDs не угадывать;
- async task без последующего status/READ не считать успехом;
- 48/48 describe не считать бизнес-успехом write;
- не смешивать Seller :8000 и Performance :8001;
- не публиковать :8001 без access-control решения.

## 11. Точная точка остановки

Пройдено:
- исследование Seller MCP и эталонной digital card;
- characteristics/image/moderation validation;
- digital stock WRITE/READ;
- точный набор 293;
- найден и доказан блокер category permission;
- Performance migrated to pinned patched 48-op runtime;
- live 48/48 catalog audit.

Продолжать:
1. Performance post-migration harmless READ;
2. Performance READ coverage;
3. fresh category permission check для 293;
4. после разрешения категории — первая партия 100;
5. затем digital posting/code/delivery;
6. затем внешний access-control Performance.

## 12. Канонические документы

- docs/PROJECT_STATE_2026-09-27.md
- server/SERVER_AUDIT_2026-09-27.md
- docs/KNOWLEDGE_BASE.md
- server/SERVER_STATE.md
- server/PERFORMANCE_RUNTIME.md
- docs/PERFORMANCE_MCP_BASELINE_2026-09-26.md
- docs/PERFORMANCE_API_MATRIX_2026-09-26.md
- docs/MASS_CATALOG_IMPORT_PLAN.md
- docs/AI_MAINTENANCE.md
- docs/TEST_HISTORY.md