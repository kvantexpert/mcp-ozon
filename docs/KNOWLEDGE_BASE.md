# Ozon MCP — база знаний для продолжения

## Главный алгоритм

DISCOVER -> READ -> VALIDATE -> WRITE -> VERIFY -> DOCUMENT

## Идентификаторы

- offer_id = seller article;
- product_id = Ozon card;
- sku = Ozon SKU;
- posting_number = конкретное digital posting.

Не считать наличие одного ID доказательством наличия остальных сущностей.

## Category workflow

category tree -> category/type -> attributes -> required -> dictionary -> payload

Category schema и category availability — разные вещи. Наличие type_id в документации не означает доступность для конкретного кабинета.

## Частые ошибки

### used_forbidden_category
Проблема доступности категории для кабинета. Сначала category tree и disabled. Не повторять массовый import.

### description_category_is_empty
Не передана обязательная description_category_id.

### 404 /v1/product/upload_digital_codes
Старый endpoint. Не использовать как рабочий механизм.

### Deprecated /v1/posting/digital/list
Проверен validation 400 Filter value is required. Для новых задач искать актуальную v2 operation.

### Dictionary error
Не угадывать numeric IDs. Делать search -> dictionary_value_id -> validate -> write.

### Async task
HTTP/task acceptance не означает бизнес-успех. Всегда task/info + READ.

### Performance 48/48
Это catalog visibility audit. Он не заменяет READ/WRITE business test.

### Performance :8001 public
Нарушение текущей архитектуры. Оставлять loopback-only до access-control решения.

## 293 import

GOODS.JSON не менять.
Точный набор: 00000002 + 00000003 = 293.
Партии: 100 + 100 + 93.
До снятия category permission blocker не импортировать.

## Digital delivery

Не считать stock=1 кодом.
Полная цепочка: CARD -> STOCK -> ORDER -> POSTING -> CODE -> DELIVERY.
Последние три стадии пока не доказаны реальным заказом.

## Credentials

Seller: `/root/.config/ozon-mcp/env`.
Performance: `/root/.config/ozon-mcp/perf.env`.
Права: 600 root:root.
Значения секретов в GitHub не записывать.

## Критерий доказанности operation

Нужны:
1. правильный operation_id;
2. правильный method/path;
3. successful call;
4. осмысленный ответ Ozon;
5. READ/verify для mutating operation;
6. запись факта в docs.

## Текущая точка

Не повторять исследование эталонной card, 48/48 catalog audit и старого digital code endpoint без новой причины.

Продолжать:
1. Performance harmless READ;
2. Performance READ coverage;
3. fresh category permission check;
4. 293 import только после разрешения category/type;
5. digital posting/code/delivery;
6. Performance external access-control.