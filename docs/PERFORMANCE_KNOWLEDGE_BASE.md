# Ozon Performance — база знаний агента

Дата: 2026-09-27

## Где находится

Отдельного GitHub-репозитория ozon-performance нет. Performance runtime является отдельным сервисом внутри kvantexpert/ozon-mcp.

- systemd: ozon-performance.service
- listen: 127.0.0.1:8001
- package: marketplaces-mcp-ru 0.6.1
- executable: /opt/kvantexpert/marketplaces-mcp-ru/.venv/bin/ozon-perf-mcp
- env: /root/.config/ozon-mcp/perf.env
- public route: отсутствует

Pinned upstream:
ec2114595695536e001e09e1144a357118852db1

## Безопасность

Performance service остается loopback-only. HTTP transport upstream не считать authentication layer. Внешний доступ не публиковать. Credentials хранятся только в perf.env.

## Работа с operations

1. Проверить docs/PERFORMANCE_API_MATRIX_2026-09-26.md.
2. Найти operation_id.
3. Проверить catalog/describe.
4. Проверить HTTP method/path/schema.
5. Сначала harmless READ.
6. WRITE только после отдельной проверки.
7. После WRITE выполнить независимый READ.
8. Зафиксировать результат.

## Типовые ошибки

### Unknown method
Проверить operation_id, установленную версию, pinned upstream и наличие patch. Не придумывать operation_id.

### 401/403
Проверить client_credentials, env, credentials, endpoint и системное время. Не печатать token.

### 8001 не отвечает
Проверить systemctl status, journalctl, ss на 8001, права perf.env, executable и restart.

### 8001 доступен из Internet
Это security problem. Вернуть loopback bind и убрать внешний route.

### 48/48 describe проходит, operation не работает
Describe подтверждает каталог/schema, а не реальное выполнение. Добавить live READ test.

### Patch исчез после обновления upstream
Обновить pinned source и reproducible patch, переустановить runtime и повторить tests. Не полагаться на ручную правку installed package.

## Следующий шаг

1. Свежий server audit.
2. harmless READ: ozon_perf_call_method(operation_id=ozonperf_get_api_client_campaign).
3. Несколько ключевых READ.
4. Зафиксировать результаты.
5. Затем проектировать минимальный AI-visible Performance tool surface.
6. Public Performance endpoint пока не открывать без отдельного access-control design.
