# AI agent handoff — Ozon MCP + Ozon Performance

Дата фиксации: 2026-09-27

## 1. Каноническая точка входа

Перед работой читать:
1. docs/PROJECT_STATE_2026-09-27.md
2. docs/KNOWLEDGE_BASE.md
3. docs/AI_AGENT_CONTEXT.md
4. docs/SETUP_AND_RECOVERY.md
5. docs/PERFORMANCE_KNOWLEDGE_BASE.md — для Performance
6. server/SERVER_AUDIT_2026-09-27.md
7. server/SERVER_STATE.md

Текущий main:
d1fc3801d3c80f6568491f5fc3f35005734577db

Важно: старые документы могут содержать исторические SHA. Фактический git HEAD проверять перед runtime-работой.

## 2. Проект

Один VPS содержит два независимых runtime:

Seller:
- 127.0.0.1:8000
- public https://ozon-mcp.kvantexpert.ru/mcp
- ozon-mcp 0.6.0

Performance:
- 127.0.0.1:8001
- public route отсутствует
- marketplaces-mcp-ru 0.6.1
- pinned upstream ec2114595695536e001e09e1144a357118852db1

Не объединять runtime и не публиковать 8001.

## 3. Что доказано

Seller:
- создание эталонной карточки;
- характеристики;
- dictionary lookup;
- изображения;
- moderation/validation;
- digital stock;
- независимый stock READ.

Performance:
- tracked 48-operation catalog;
- 48/48 describe audit;
- safety corrections for semantic READ/WRITE classification;
- systemd deployment;
- loopback-only runtime.

## 4. Текущий blocker

Массовый импорт 293 позиций НЕ выполнен.

Набор:
- 00000002 -> 193
- 00000003 -> 100
- total 293

План batch:
100 + 100 + 93.

Последний blocker:
- category/type returned disabled=true;
- import without category -> description_category_is_empty;
- import with 200001489 -> used_forbidden_category.

Не повторять mass import, пока свежий category tree не покажет допустимые category/type.

## 5. Следующая работа

1. Свежий server audit.
2. Harmless Performance READ.
3. Key Performance READ coverage.
4. Свежий category tree / limits / product list.
5. Если category allowed — подготовить первую batch <=100.
6. Import -> task/status -> independent READ.
7. Следующие batches только после успешной проверки первой.

Digital delivery отдельно:
- не считать stock=1 наличием digital code;
- не использовать stale /v1/product/upload_digital_codes;
- posting-based code upload требует реального posting_number.

## 6. Основные ошибки

- used_forbidden_category: refresh category tree, не повторять старый category.
- description_category_is_empty: в payload отсутствует category.
- dictionary ID guessed: search dictionary -> exact value -> numeric ID -> write.
- WRITE через read-only tool: использовать write method + confirmation.
- async task accepted != business success: обязательно task/status + READ.
- 404 stale endpoint: заново describe/search current API.
- GET может быть WRITE, POST может быть READ: safety определять по semantics.
- 48/48 describe != бизнес-успех всех operations.
- 401: проверять credentials/auth, не отключать security.
- 8001 наружу: stop condition, вернуть loopback.

Подробности: docs/KNOWLEDGE_BASE.md и docs/PERFORMANCE_KNOWLEDGE_BASE.md.

## 7. Security

Secrets:
- /root/.config/ozon-mcp/env
- /root/.config/ozon-mcp/perf.env

Ожидается mode 600 root:root.

Не выводить значения secrets, не класть в GitHub.

## 8. Рабочий принцип агента

DISCOVER -> READ -> VALIDATE -> WRITE -> TASK/STATUS -> VERIFY -> DOCUMENT

Для любого нового Ozon operation сначала найти/описать method и проверить safety. WRITE выполнять только в явном write-контуре.

## 9. Stop conditions

Не продолжать функциональный WRITE, если:
- service inactive;
- 8001 exposed;
- credentials permissions broken;
- category disabled;
- runtime catalog расходится с pinned/patch docs;
- endpoint неожиданно стал 404/403/410;
- working tree содержит неожиданные изменения.

## 10. Связь с другими проектами

- 1C Gateway: https://github.com/kvantexpert/vm-mcp-1c
- Mini App/Bitrix: https://github.com/kvantexpert/kvantexpert-app
- Site: https://github.com/kvantexpert/kvantexpert-site
- Procurement MCP: https://github.com/kvantexpert/mcp-zakupki
