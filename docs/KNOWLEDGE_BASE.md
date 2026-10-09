# Ozon MCP — база знаний

Дата актуализации: 2026-09-27

Это основной практический справочник проекта: архитектура, настройка, ошибки, причины, исправления, безопасность и порядок продолжения.

Перед любой новой работой:
1. прочитать этот файл;
2. прочитать `docs/PROJECT_STATE_2026-09-27.md`;
3. сверить текущий Git HEAD и server state;
4. выполнить READ перед WRITE.

## 1. Архитектура

Один VPS содержит два независимых MCP runtime.

```text
Internet
   |
   +--> nginx HTTPS
   |      |
   |      +--> Seller MCP 127.0.0.1:8000
   |              |
   |              +--> Ozon Seller API
   |
   +--> Performance MCP
          127.0.0.1:8001
             |
             +--> OAuth client_credentials
             |
             +--> Ozon Performance API
```

Public Seller MCP:
`https://ozon-mcp.kvantexpert.ru/mcp`

Performance MCP:
- public route отсутствует;
- `127.0.0.1:8001` должен оставаться loopback-only.

Production server:
- host: `cv7976275`
- OS: Ubuntu 22.04.4 LTS
- Seller service: `ozon-mcp.service`
- Performance service: `ozon-performance.service`
- Seller backend: `127.0.0.1:8000`
- Performance backend: `127.0.0.1:8001`

## 2. Runtime versions

Seller:
- `ozon-mcp-ru 0.6.0`
- запуск: `/root/.local/bin/uvx --from 'ozon-mcp-ru==0.6.0' ozon-mcp-ru`
- env: `/root/.config/ozon-mcp/env`

Performance:
- `marketplaces-mcp-ru 0.6.1`
- pinned upstream: `ec2114595695536e001e09e1144a357118852db1`
- runtime: `/opt/kvantexpert/marketplaces-mcp-ru/.venv/bin/ozon-perf-mcp`
- env: `/root/.config/ozon-mcp/perf.env`

Env files:
- expected mode: `600 root:root`
- secrets никогда не помещать в GitHub.

## 3. Главный принцип работы

Для всех новых операций:

```text
DISCOVER
  -> READ
  -> VALIDATE
  -> WRITE
  -> TASK/STATUS
  -> VERIFY
  -> DOCUMENT
```

Нельзя начинать исследование с WRITE.

Для новой operation:
1. search_methods;
2. describe_method;
3. проверить method/path/safety/schema;
4. READ, если возможно;
5. WRITE только с подтверждением;
6. проверить task/status;
7. независимый READ;
8. записать результат.

## 4. Seller MCP — сущности

Различать:

- `offer_id` — артикул продавца;
- `product_id` — карточка Ozon;
- `sku` — SKU;
- `posting_number` — конкретное цифровое отправление.

Наличие одного ID не доказывает наличие остальных.

Эталонная подтверждённая карточка:

- offer_id: `4601546116680`
- product_id: `6417979753`
- sku: `5865629857`
- category: `200001489`
- type_id: `971075562`
- name: `1С:Бухгалтерия 8 ПРОФ. Электронная поставка`
- price: 23000 RUB
- VAT: 0

## 5. Card lifecycle

```text
INPUT
  -> CATEGORY
  -> TYPE
  -> ATTRIBUTES
  -> DICTIONARIES
  -> CREATE
  -> CHARACTERISTICS
  -> IMAGES
  -> PRICE
  -> MODERATION/VALIDATION
  -> DIGITAL STOCK
  -> ORDER
  -> POSTING
  -> DIGITAL CODE
  -> DELIVERY
  -> VERIFY
```

Фактически доказано:
`CARD -> STOCK`

Не доказано реальным заказом:
`ORDER -> POSTING -> CODE -> DELIVERY`

## 6. Category / attributes

Эталонная исследованная категория:
- `description_category_id=200001489`
- `type_id=971075562`

Важно:
эти ID нельзя считать доступными новому аккаунту без свежего READ.

Category workflow:

```text
category tree
 -> category/type
 -> attributes
 -> required/optional
 -> dictionary
 -> payload
```

Обязательным для эталона был:
- attribute `8229` — Тип.

Подтверждённые dictionary values:
- 8229 -> Код активации офисного приложения -> 971075562
- 85 -> 1С -> 970871978
- 77 -> 1С -> 5058512
- 9279 -> Бухгалтерия -> 87549223
- 5162 -> Windows -> 29934
- 11469 -> Русская версия -> 970831215
- 74 -> 1С -> 5058512

Не угадывать:
- territory;
- activation territory;
- usage territory;
- license term;
- activation term;
- device count;
- edition;
- OS version.

Dictionary workflow:

```text
text
 -> search dictionary
 -> dictionary_value_id
 -> validate
 -> write
```

## 7. Подтвержденные Seller операции

Создание:
- `POST /v3/product/import`
- async
- task_id
- максимум 100 items/request.

Статус импорта:
- `POST /v1/product/import/info`

Характеристики:
- `POST /v1/product/attributes/update`
- WRITE
- через `ozon_write_method` + confirmation.

Изображения:
- `POST /v1/product/pictures/import`

Digital stock:
- `POST /v1/product/digital/stocks/import`

Независимый stock READ:
- `POST /v4/product/info/stocks`

Для эталона подтверждено:
- WRITE stock -> 1
- независимый READ -> present=1, reserved=0, sku=5865629857.

## 8. Digital delivery — что уже выяснено

Актуальный исследованный список:
- `POST /v2/posting/digital/list` — READ, напрямую 200, но текущий MCP catalog этот v2 operation не содержит;
- `POST /v1/posting/digital/list` — deprecated, пустой body -> 400 `Filter value is required`;
- `POST /v1/posting/digital/codes/upload` — WRITE для конкретного posting;
- `POST /v1/product/upload_digital_codes` — проверен и получил 404, считать stale.

Не вызывать `/v1/posting/digital/codes/upload` без реального `posting_number`.

Не считать `stock=1` наличием конкретного digital code.

Реальные digital codes не писать в GitHub, docs или обычные logs.

## 9. Ошибки Seller и исправления

### 9.1 `description_category_is_empty`

Симптом:
import task = failed, ошибка `description_category_is_empty`.

Причина:
в payload отсутствует обязательная category.

Исправление:
получить актуальный category/type и передать `description_category_id`.

### 9.2 `used_forbidden_category`

Симптом:
import с `description_category_id=200001489` завершился `used_forbidden_category`.

Причина:
категория запрещена/недоступна конкретному аккаунту.

Исправление:
1. сделать свежий READ category tree;
2. проверить `disabled`;
3. найти разрешённую category/type;
4. не повторять массовый import со старой категорией.

Это был исторический blocker массового импорта 293 на предыдущем этапе. Сейчас этап импорта закрыт; повторно этот workflow не запускать без явного запроса.

### 9.3 Исторический статус 293

Последний документированный новый аккаунт:
- products: 0/500;
- daily_create: 2/1500;
- daily_update: 0/20000;
- rate: 30000/min.

Вывод:
лимит ассортимента не был blocker.

Правильная историческая диагностика:
category tree -> disabled -> import test -> task/status.

**Текущий статус:** 169 позиций уже импортированы. Этап импорта закрыт. Старые лимиты и диагностика выше не являются текущим планом работ.

### 9.4 WRITE выполнен read-only MCP tool

Из истории:
`ozon_update_characteristics` был остановлен safety gate через read-only `ozon_call_method`.

Исправление:
использовать `ozon_write_method` + `confirm_write=true`.

### 9.5 Угадывание dictionary IDs

Причина:
текстовое значение не гарантирует numeric dictionary_value_id.

Исправление:
search -> exact value -> ID -> validation -> write.

### 9.6 Async task принят, но бизнес-операция не проверена

HTTP/task acceptance не означает успех.

Исправление:
task/status -> ошибки -> повторный READ.

### 9.7 `/v1/product/upload_digital_codes` -> 404

Подтверждено и через MCP, и напрямую с VPS.

Причина:
endpoint устарел на стороне текущего Ozon API.

Исправление:
не повторять. Для доставки искать posting-based workflow и актуальные v2 operation.

### 9.8 `/v1/posting/digital/list` -> 400

Ошибка:
`Filter value is required`.

Причина:
deprecated endpoint + обязательный filter contract.

Исправление:
не строить новую реализацию на v1; исследовать актуальный v2.

### 9.9 Physical attributes заполнены «для полноты»

Для электронной поставки нельзя автоматически подставлять physical carrier/packing/media.

Исправление:
optional поле оставлять пустым, пока источник не подтверждает реальное значение.

### 9.10 Stock=1 принят за выданный код

Причина:
смешение сущностей stock и code.

Исправление:
проверять posting_number, required quantity, code upload и delivery отдельно.

## 10. Исторический 293 catalog

Этот раздел описывает исходный набор 293, использованный на историческом этапе. Он не является текущей задачей и не должен запускать новый импорт.

Источник:
`GOODS.JSON`, не изменять.

Точный набор:
- group `00000002` -> 193
- group `00000003` -> 100
- total 293

Исключены:
- 00000292 -> 28
- 00000296 -> 4
- 00000314 -> 2

Партии:
`100 + 100 + 93`

Правило цены:
1. 00001;
2. иначе 00003;
3. иначе 00004.

Подтверждение:
- 244 товара -> 00001;
- 38 -> 00003+00004;
- 6 -> 00002+00003+00004;
- 5 -> only 00004.

Базовое правило:
не смешивать Basic/PROF/CORP и другие предложения автоматически.

## 11. Исторический алгоритм массового импорта

**Справочно. Этап закрыт: 169 позиций уже импортированы. Не выполнять эти действия автоматически и не возвращаться к ним без явного запроса.**

До снятия category blocker:
- не отправлять 293;
- не создавать большую batch;
- не повторять forbidden category.

После снятия:
1. fresh READ category tree;
2. проверить disabled=false;
3. построить первую batch <=100;
4. локально проверить payload;
5. WRITE;
6. сохранить task_id;
7. import/info;
8. зафиксировать ошибки;
9. только затем следующая batch.

## 12. Performance MCP

Отдельный runtime:
- package marketplaces-mcp-ru 0.6.1;
- pinned commit ec2114595695536e001e09e1144a357118852db1;
- local runtime /opt/kvantexpert/marketplaces-mcp-ru/.venv/bin/ozon-perf-mcp;
- backend 127.0.0.1:8001;
- public route отсутствует.

Tracked catalog:
`patches/marketplaces-mcp-ru/perf_endpoints.yaml`

Catalog:
**48 operation_id**

Live audit 26.09.2026:
**48/48 found; 0 FAIL**

Но:
48/48 describe != бизнес-успех всех operations.
Это проверка каталога/dispatcher visibility.

## 13. Performance safety corrections

Подтверждены:
- POST /api/client/min/sku -> read
- POST /api/client/search_promo/bids/recommendation -> read
- GET all SKU promo activate -> write
- GET all SKU promo deactivate -> write
- GET all SKU promo set_bid -> write

Добавлены:
- POST /api/client/statistics/products/sku -> read
- PATCH /api/client/campaign/{campaignId} -> write
- GET /api/client/campaign/all_sku_promo/set_bid -> write

Нельзя определять safety только по HTTP verb.

## 14. Performance ошибки и исправления

### 14.1 Считать 48 operation_id отдельными MCP tools

Ошибка модели проекта.

Реальность:
generic dispatcher:
- `ozon_perf_call_method`
- `ozon_perf_write_method`
- `ozon_perf_delete_method`

Исправление:
работать через operation_id и safety class.

### 14.2 GET считать read-only

Ошибка:
GET может менять состояние.

Исправление:
смотреть semantic behavior. Activate/deactivate/set_bid классифицированы как write.

### 14.3 POST считать write

Ошибка:
POST может только читать.

Исправление:
`min/sku` и `search_promo/bids/recommendation` классифицированы read.

### 14.4 48/48 describe считать готовностью всех API

Ошибка:
describe подтверждает только наличие описания и dispatch contract.

Исправление:
отдельно проводить harmless READ smoke и затем контролируемые WRITE.

### 14.5 Performance сделать публичным сразу

Не делать.

До public route:
1. READ coverage;
2. minimal AI-visible tool set;
3. write confirmation;
4. access-control design;
5. только потом nginx/public.

## 15. Server errors и recovery

### service inactive

Проверить:

```bash
systemctl status ozon-mcp.service --no-pager
journalctl -u ozon-mcp.service -n 100 --no-pager

systemctl status ozon-performance.service --no-pager
journalctl -u ozon-performance.service -n 100 --no-pager
```

Потом:
- проверить env;
- проверить unit;
- py/runtime;
- restart только после понимания ошибки.

### port 8000/8001 unavailable

Проверить:

```bash
ss -lntp | grep -E ':8000|:8001' || true
```

Ожидание:
- 8000 loopback;
- 8001 loopback.

### Performance 8001 exposed

Это security incident/stop condition.

Не добавлять public nginx route. Вернуть loopback bind, затем проверить firewall/nginx.

### nginx broken

```bash
nginx -t
journalctl -u nginx -n 100 --no-pager
systemctl status nginx --no-pager
```

Сначала исправить config, затем reload.

### credentials problem

Проверять только:
```bash
stat -c '%a %U:%G %n' /root/.config/ozon-mcp/env /root/.config/ozon-mcp/perf.env
```

Не печатать содержимое.

### Seller public MCP 401

Проверять nginx/Bearer configuration и client Authorization header.

Не отключать auth для диагностики.

## 16. Проверка Ozon API

Если endpoint перестал работать:

1. search_methods;
2. describe_method;
3. проверить current method/path;
4. выполнить harmless READ;
5. только потом менять integration.

Не использовать stale endpoint только потому, что он есть в старой документации.

## 17. Recovery checklist

```text
[ ] Git HEAD expected
[ ] working tree understood
[ ] Seller service active
[ ] Performance service active
[ ] 8000 loopback
[ ] 8001 loopback
[ ] env permissions 600
[ ] nginx -t OK
[ ] Seller MCP initialize OK
[ ] Performance initialize OK
[ ] 48/48 Performance describe OK
[ ] harmless Performance READ OK
[ ] fresh category tree READ done — only when current category work is requested
[ ] category disabled status known — only when current category work is requested
[ ] no secrets in output
```

## 18. Stop conditions

Сначала расследовать, а не делать функциональные WRITE, если:
- 8001 опубликован;
- service inactive;
- nginx -t fails;
- env permissions != 600;
- category disabled — только если выполняется явно запрошенная текущая работа с категорией;
- import returns forbidden category — только если явно запрошен текущий import workflow;
- API endpoint unexpectedly returns 404/410;
- operation safety unexpectedly changes;
- runtime catalog differs from tracked patch;
- Git working tree contains unexpected changes.

## 19. Канонические документы

- docs/PROJECT_STATE_2026-09-27.md — точка продолжения
- docs/KNOWLEDGE_BASE.md — эта база знаний
- docs/SETUP_AND_RECOVERY.md — настройка и recovery
- server/SERVER_AUDIT_2026-09-27.md — production audit
- server/SERVER_STATE.md — VPS state
- server/VPS_RUNTIME.md — Seller runtime
- server/PERFORMANCE_RUNTIME.md — Performance runtime
- docs/OZON_OPERATIONS.md — operation map
- docs/TEST_HISTORY.md — фактические тесты
- docs/MASS_CATALOG_IMPORT_PLAN.md — исторический план 293, не текущая задача
- docs/PERFORMANCE_API_MATRIX_2026-09-26.md — 48 Performance ops
- docs/AI_MAINTENANCE.md — правила для будущего AI
