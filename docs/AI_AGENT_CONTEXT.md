# AI agent context — Ozon MCP

Дата: 2026-09-27

## Читать перед работой

1. docs/PROJECT_STATE_2026-09-27.md — текущее состояние.
2. docs/KNOWLEDGE_BASE.md — правила, ошибки и подтвержденные операции.
3. server/SERVER_STATE.md — состояние VPS.
4. server/SERVER_AUDIT_2026-09-27.md — production audit.
5. docs/OZON_OPERATIONS.md — Seller API.
6. docs/PERFORMANCE_API_MATRIX_2026-09-26.md — Performance API.
7. docs/PERFORMANCE_KNOWLEDGE_BASE.md — Performance runtime.

## Архитектура

Seller MCP: 127.0.0.1:8000 -> https://ozon-mcp.kvantexpert.ru/mcp
Performance MCP: 127.0.0.1:8001 -> public route отсутствует

Не объединять runtime без отдельного решения.

## Правильный цикл

DISCOVER -> READ -> VALIDATE -> WRITE -> TASK/STATUS -> VERIFY -> DOCUMENT

Любой WRITE начинать только после READ/validation и проверки schema.

## Текущая точка

Seller:
- эталонная digital card проверена;
- характеристики, dictionary values, image, moderation, validation и digital stock 0 -> 1 подтверждены;
- реальный ORDER -> POSTING -> CODE -> DELIVERY не подтвержден.

293:
- набор определен;
- массовый import не выполнен;
- последний blocker: disabled category/type и used_forbidden_category;
- не повторять массовый import без свежего READ category tree.

Performance:
- runtime 0.6.1;
- pinned upstream зафиксирован;
- 48 операций catalog и 48/48 describe проверены;
- это подтверждает каталог/schema, но не бизнес-функциональность каждой операции;
- следующий шаг: harmless READ coverage.

## Жесткие правила

- секреты Ozon не попадают в Git/docs/logs;
- реальные digital codes не записывать в docs;
- не путать offer_id, product_id, sku, posting_number;
- не делать WRITE только потому, что operation существует;
- не публиковать Performance :8001 наружу;
- upstream patch должен быть воспроизводимым.

## Связанные проекты

- 1C Gateway: https://github.com/kvantexpert/vm-mcp-1c
- Site: https://github.com/kvantexpert/kvantexpert-site
- Mini App/Bitrix: https://github.com/kvantexpert/kvantexpert-app
- Procurement MCP: https://github.com/kvantexpert/mcp-zakupki
