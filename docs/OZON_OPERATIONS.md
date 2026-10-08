# Карта Ozon operations

Этот документ описывает операции, обнаруженные в ходе проекта. Названия operation_id относятся к каталогу Ozon MCP, а endpoint — к Ozon Seller API.

## 1. Категории

### Category tree

Используется для определения:

- description_category_id;
- type_id;
- доступности категории.

Для эталона:

- 200001489 — Цифровые товары;
- 971075562 — Код активации офисного приложения.

### Category attributes

Получение актуальных характеристик категории.

Использовать перед созданием новой карточки.

## 2. Dictionary

### ozon_search_attributes_dictionary

Endpoint:

`POST /v1/description-category/attribute/values/search`

Назначение:

поиск dictionary_value_id.

READ по смыслу.

### ozon_attributes_dictionary

Endpoint:

`POST /v1/description-category/attribute/values`

Каталог помечает operation как write, хотя семантически это получение словаря. Не вызывать вслепую; сначала describe и проверять актуальный контракт.

## 3. Создание/изменение

### Product import

`POST /v3/product/import`

Создание/импорт карточки.

### ozon_update_characteristics

`POST /v1/product/attributes/update`

WRITE.

Через `ozon_call_method` MCP блокирует operation. Использовать:

`ozon_write_method + confirm_write=true`

После выполнения проверять task/status и читать карточку.

### ozon_product_pictures_import

`POST /v1/product/pictures/import`

WRITE.

Используется для images[] / images360 / color_image.

## 4. Digital stock

### ozon_post_v1_product_digital_stocks_import

`POST /v1/product/digital/stocks/import`

WRITE.

Тело:

```json
{
  "stocks": [
    {
      "offer_id": "4601546116680",
      "stock": 1
    }
  ]
}
```

В тесте:

`updated=true`

## 5. Stock read

### product info stocks

`POST /v4/product/info/stocks`

Использован для независимой проверки.

Для эталона получил:

- present=1;
- reserved=0;
- type=fbo;
- sku=5865629857.

## 6. Digital postings

### v2/posting/digital/list

`POST /v2/posting/digital/list`

READ.

Request supports:

- order_numbers;
- posting_numbers;
- since;
- to;
- cursor;
- limit;
- sort_dir;
- with.analytics_data;
- with.financial_data;
- with.legal_info.

В текущем MCP operation catalog эта v2 operation не была найдена, хотя endpoint существует и напрямую отвечает 200.

Прямой тест за 2026-09-01—2026-09-22:

`postings=[]`

### v1/posting/digital/list

`POST /v1/posting/digital/list`

В каталоге MCP присутствует как deprecated READ operation.

Пустой body дал 400:

`Filter value is required`

## 7. Digital code upload

### Старый endpoint

`POST /v1/product/upload_digital_codes`

Operation:

`ozon_upload_activation_codes`

Ожидаемая модель ранее была:

```json
{
  "digital_codes": ["CODE"],
  "product_id": 6417979753
}
```

Фактический текущий Ozon API ответил 404 даже при прямом запросе с VPS.

Статус:

**НЕ ИСПОЛЬЗОВАТЬ как рабочий механизм.**

### Posting code upload

`POST /v1/posting/digital/codes/upload`

Operation:

`ozon_post_v1_posting_digital_codes_upload`

Это WRITE для конкретного posting.

Модель:

```json
{
  "posting_number": "0001-1",
  "exemplars_by_sku": [
    {
      "sku": 5865629857,
      "exemplar_qty": 1,
      "not_available_exemplar_qty": 0,
      "exemplar_keys": ["CODE-1"]
    }
  ]
}
```

Не использовать без реального posting_number.

## 8. Safety

Для каждой operation сначала выполнять describe.

Проверять:

- method;
- path;
- safety;
- params;
- doc;
- pagination.

WRITE не запускать через read-only tool.

## 9. Операционная таблица

| Операция | Endpoint | Тип | Назначение |
|---|---|---|---|
| category tree | category API | READ | найти категорию/type |
| category attributes | category API | READ | получить attributes |
| ozon_search_attributes_dictionary | /v1/description-category/attribute/values/search | READ | найти dictionary value |
| product import | /v3/product/import | WRITE | создать товар |
| ozon_update_characteristics | /v1/product/attributes/update | WRITE | изменить характеристики |
| ozon_product_pictures_import | /v1/product/pictures/import | WRITE | изображения |
| product info stocks | /v4/product/info/stocks | READ | проверить остаток |
| digital stocks import | /v1/product/digital/stocks/import | WRITE | цифровой остаток |
| v2 digital list | /v2/posting/digital/list | READ | цифровые postings |
| v1 digital list | /v1/posting/digital/list | READ/DEPRECATED | старый список |
| posting digital codes upload | /v1/posting/digital/codes/upload | WRITE | код конкретному posting |
| upload activation codes | /v1/product/upload_digital_codes | WRITE/STALE | текущий API 404 |

## 10. Как расширять карту

Новая operation добавляется только после:

1. search_methods;
2. describe_method;
3. проверки endpoint;
4. проверки request schema;
5. READ-теста, если возможно;
6. WRITE-теста только с безопасными данными;
7. фиксации результата в TEST_HISTORY.


## 11. Канонический WRITE-шаблон создания карточки

Для пользовательского триггера «создай карточку» использовать operation:

`ozon_product_import`

MCP-вызов:

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

Назначение:

- `ozon_write_method` — WRITE-диспетчер;
- `ozon_product_import` — операция создания/импорта карточки;
- `body.items` — список товаров для импорта;
- `confirm_write=True` — обязательное подтверждение WRITE.

После вызова:

`task_id → status/errors → READ карточки`

Последний подтверждённый E2E:

- `task_id = 5772118688`;
- `offer_id = 2900002078329`;
- `195 BYN`;
- `category_id = 46590429`;
- `type_id = 392638731`.

Не считать эти значения шаблоном нового товара. Они являются контрольным примером успешного запуска.


## 12. Подтверждённый payload-контракт /v3/product/import для OZON-BY

Для каждого элемента body.items[] поля размеров и веса передаются на верхнем уровне item:

{
  offer_id, name, description_category_id, type_id, price, vat, currency_code,
  images, primary_image,
  depth, width, height, dimension_unit, weight, weight_unit,
  attributes
}

Нельзя использовать вложенный объект dimensions. Такая форма приводит к missing_dimension.

Обязательный контроль перед массовым WRITE:

1. offer_id и цена взяты из актуального BY-реестра.
2. description_category_id/type_id соответствуют актуальной разрешённой категории.
3. 8229 заполнен актуальным dictionary value.
4. 22232 (ТН ВЭД) заполнен применимым dictionary value.
5. depth/width/height/dimension_unit/weight/weight_unit находятся непосредственно в item.
6. После WRITE: task_id → ozon_product_import_info → errors=[].

Подтверждение 2026-10-08: массовая задача 5772274745 — 57 товаров, 57/57 imported, errors=[]. Вместе с контрольными товарами реестр PREVIEW_61 доведён до 61/61 imported, errors=[].
