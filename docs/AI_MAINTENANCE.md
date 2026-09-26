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
