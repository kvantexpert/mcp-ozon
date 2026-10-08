# Инструкция для AI-ассистента

Этот документ предназначен для будущего AI, который продолжит проект через месяц или год.

## 1. Сначала прочитать

Перед изменениями сначала прочитать:

1. `docs/PROJECT_STATE_2026-09-27.md`
2. `server/SERVER_AUDIT_2026-09-27.md`
3. `docs/KNOWLEDGE_BASE.md`
4. README.md
2. docs/PROJECT_MODEL.md
3. docs/CARD_LIFECYCLE.md
4. docs/ATTRIBUTES_AND_DICTIONARIES.md
5. docs/OZON_OPERATIONS.md
6. docs/DIGITAL_DELIVERY.md
7. docs/TEST_HISTORY.md

Затем проверить актуальное состояние репозитория и сервера.

## 2. Не считать документацию вечной

Документация — база знаний, но Ozon API может измениться.

Если operation кажется устаревшей:

1. search_methods;
2. describe_method;
3. проверить endpoint;
4. проверить request schema;
5. только после этого изменять код.

## 3. Никогда не начинать с WRITE

Правильный порядок:

`SEARCH → DESCRIBE → READ → VALIDATE → WRITE → VERIFY`

WRITE без необходимости не выполнять.

## 4. Для новой карточки

AI должен:

1. получить входные данные;
2. определить товар как digital;
3. найти категорию;
4. найти type_id;
5. получить attributes;
6. определить required;
7. найти dictionary values;
8. проверить лицензионные поля;
9. создать товар;
10. сохранить product_id/sku;
11. записать attributes;
12. загрузить images;
13. установить price;
14. проверить moderation/validation;
15. установить digital stock;
16. перечитать карточку.

## 5. Не угадывать

Особенно нельзя угадывать:

- territory;
- activation territory;
- usage territory;
- license term;
- activation term;
- device count;
- edition;
- OS version.

Если нет достоверного источника:

`не заполнять`

## 6. Dictionary workflow

Нельзя:

`"Windows" → записать строку`

если поле dictionary-controlled.

Нужно:

`"Windows" → search → dictionary_value_id → validate → write`

## 7. ID discipline

Всегда различать:

- offer_id;
- product_id;
- sku;
- posting_number.

Перед операцией проверять, какой именно идентификатор требуется.

## 8. WRITE discipline

Для MCP:

- read operations через read tool;
- write operations через `ozon_write_method`;
- обязательно подтверждение WRITE;
- после WRITE проверить task/status;
- затем выполнить READ.

## 9. Не повторять уже выполненные опасные тесты

Если TEST_HISTORY показывает успешный WRITE, не повторять его без необходимости.

Особенно:

- не загружать повторно изображение без причины;
- не менять stock случайно;
- не отправлять настоящий digital code;
- не вызывать deprecated endpoint.

## 10. Credentials

Никогда не просить пользователя прислать:

- Client-Id;
- Api-Key.

Использовать серверный env.

Не коммитить credentials.

Не выводить credentials в логи.

## 11. Digital delivery

Не считать `stock=1` доказательством наличия конкретного кода.

Полная цепочка должна быть:

`CARD → STOCK → ORDER → POSTING → CODE → DELIVERY`

Последние три стадии пока не доказаны реальным заказом.

## 12. Изменение документации

После существенного теста обновить:

- TEST_HISTORY.md — фактический результат;
- OZON_OPERATIONS.md — новую operation;
- CARD_LIFECYCLE.md — изменение алгоритма;
- DIGITAL_DELIVERY.md — изменения digital flow;
- README.md — если изменился основной механизм.

## 13. Что писать в историю

Каждый тест должен фиксировать:

- дата;
- operation_id;
- endpoint;
- READ/WRITE;
- входные ключевые параметры без секретов;
- результат;
- product_id;
- offer_id;
- sku;
- task_id;
- ошибки;
- вывод;
- что делать дальше.

Не записывать реальные API keys или реальные digital codes.

## 14. Как продолжить проект через год

Начать не с предположений.

Выполнить:

```
git status
git log --oneline -20
прочитать docs/
проверить текущий MCP endpoint
проверить Ozon operation catalog
проверить product_id / offer_id / sku
сделать READ
сравнить с TEST_HISTORY
```

Если текущее состояние отличается от документации — сначала обновить документацию.

## 15. Главный принцип

AI должен поддерживать воспроизводимость:

`KNOW WHAT EXISTS → KNOW WHAT CHANGES → CHANGE ONE THING → VERIFY → DOCUMENT`

Не переписывать архитектуру только потому, что появился новый endpoint.

Сначала доказать новый механизм тестом, затем закрепить его в документации.


## 16. Канонический триггер «создай карточку»

Если пользователь говорит:

- «создай карточку»;
- «создай товар на Ozon»;
- «создай карточку Ozon»;

не начинать повторное исследование уже подтвержденного механизма создания карточки.

Для WRITE использовать канонический вызов:

```python
await session.call_tool(
    "ozon_write_method",
    arguments={
        "operation_id": "ozon_product_import",
        "body": {
            "items": [PRODUCT]
        },
        "confirm_write": True
    }
)
```

Правила:

1. `PRODUCT` формируется из актуальных входных данных конкретного товара.
2. `confirm_write=True` обязателен.
3. После WRITE сохранить возвращённый `task_id`.
4. Проверить статус задачи и ошибки.
5. Выполнить READ для подтверждения созданной карточки.
6. Не повторять исследование способа вызова, если operation и schema не изменились.
7. Если operation/schema/endpoint изменились или вызов перестал работать — вернуться к `SEARCH → DESCRIBE → READ → VALIDATE` и обновить канон.
8. Реальные credentials и секреты в документацию не записывать.

Последний подтверждённый E2E-тест этого шаблона:

- `task_id = 5772118688`
- `offer_id = 2900002078329`
- цена = `195 BYN`
- `category_id = 46590429`
- `type_id = 392638731`

Это шаблон запуска, а не готовый payload товара: обязательные поля `PRODUCT` должны формироваться из конкретной карточки и актуальной схемы Ozon.


## 17. Подтверждённый шаблон BY-карточки после импорта 61 товара

Для OZON-BY после E2E-проверки подтверждено:

- description_category_id = 46590429;
- type_id = 392638731;
- depth = 20, width = 20, height = 20;
- dimension_unit = mm;
- weight = 10;
- weight_unit = g;
- обязательный атрибут 22232 (ТН ВЭД) должен быть заполнен;
- в /v3/product/import размеры и вес передаются не вложенным объектом dimensions, а полями непосредственно внутри каждого item.

Критически важно: не использовать конструкцию dimensions: {...} для /v3/product/import — Ozon принимает WRITE, но затем возвращает missing_dimension.

Проверенный контрольный 22232 для тестовой схемы: dictionary_value_id = 971399739 (8467890000 - Прочие инструменты). Для новых товаров значение ТН ВЭД нельзя копировать автоматически без проверки применимости.

Контроль массового импорта 2026-10-08:

- 2900002078329 — imported;
- 2900002006896 — imported, errors=[];
- 2900002006902 — imported, errors=[];
- 2900002006919 — imported, errors=[];
- массовая задача 5772274745 — 57/57 imported, errors=[];
- итог реестра из 61 товара — 61/61 imported, errors=[].
