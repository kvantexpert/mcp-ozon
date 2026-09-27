# AI agent context — Ozon MCP

Дата: 2026-09-27

**Первая точка входа:** docs/AI_AGENT_HANDOFF_2026-09-27.md

## Читать перед работой

1. docs/AI_AGENT_HANDOFF_2026-09-27.md
2. docs/PROJECT_STATE_2026-09-27.md
3. docs/KNOWLEDGE_BASE.md
4. server/SERVER_AUDIT_2026-09-27.md
5. server/SERVER_STATE.md
6. docs/OZON_OPERATIONS.md
7. docs/PERFORMANCE_KNOWLEDGE_BASE.md
8. docs/PERFORMANCE_API_MATRIX_2026-09-26.md

Git SHA в старых документах может быть историческим. Перед runtime-работой всегда проверять фактический main/working tree.

## Архитектурные границы

Seller MCP: 127.0.0.1:8000 -> https://ozon-mcp.kvantexpert.ru/mcp

Performance MCP: 127.0.0.1:8001 -> public route отсутствует

Seller и Performance — независимые runtime. 8001 не публиковать.

## Правило агента

DISCOVER -> READ -> VALIDATE -> WRITE -> TASK/STATUS -> VERIFY -> DOCUMENT

Любой WRITE начинать только после READ/validation и проверки schema/safety.

## Текущая точка

Seller:
- эталонная digital card проверена;
- characteristics, dictionaries, image, moderation, validation и digital stock подтверждены;
- массовый импорт 293 позиций пока BLOCKED старой/недоступной category.

Performance:
- 0.6.1 runtime;
- pinned upstream;
- 48 operation_id tracked;
- 48/48 describe audit;
- safety classification исправлена;
- следующий шаг — harmless READ coverage.

## Важные документы

Seller ошибки и рабочий процесс: docs/KNOWLEDGE_BASE.md

Performance ошибки и рабочий процесс: docs/PERFORMANCE_KNOWLEDGE_BASE.md

Server/runtime audit: server/SERVER_AUDIT_2026-09-27.md

Recovery: docs/SETUP_AND_RECOVERY.md

## Hard rules

- secrets only in server env, never GitHub;
- 8001 loopback-only;
- do not retry 293 mass import while category is disabled;
- do not use stale digital endpoints;
- do not treat stock as digital code;
- do not infer safety from HTTP verb;
- 48/48 describe is not proof of business success;
- do not expose Performance publicly without a separate access-control design;
