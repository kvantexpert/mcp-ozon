# Ozon MCP — аудит production VPS

Дата документа: 2026-09-27.

Важно: это фиксация последнего доказанного состояния. В этом сообщении VPS не опрашивался напрямую заново. Последний документированный live-аудит Performance выполнен 26.09.2026; Seller/293 checks — 24.09.2026.

## Safe audit

```bash
set -e
hostname
hostname -I
lsb_release -ds 2>/dev/null || true

systemctl is-enabled ozon-mcp.service
systemctl is-active ozon-mcp.service
systemctl show ozon-mcp.service -p ActiveState -p SubState -p MainPID -p ExecMainStartTimestamp

systemctl is-enabled ozon-performance.service
systemctl is-active ozon-performance.service
systemctl show ozon-performance.service -p ActiveState -p SubState -p MainPID -p ExecMainStartTimestamp

ss -lntp | grep -E ':8000|:8001' || true
nginx -t

stat -c '%a %U:%G %n' /root/.config/ozon-mcp/env /root/.config/ozon-mcp/perf.env
```

Не печатать содержимое env-файлов.

## Expected

- Seller :8000 = loopback;
- Performance :8001 = loopback;
- nginx :80/:443 = public entry;
- env files = 600 root:root;
- оба systemd service = active;
- nginx -t = successful.

## Seller audit

```bash
systemctl status ozon-mcp.service --no-pager
journalctl -u ozon-mcp.service -n 100 --no-pager
systemctl cat ozon-mcp.service
```

Ожидаемый запуск:
`/root/.local/bin/uvx --from 'ozon-mcp-ru==0.6.0' ozon-mcp-ru`

## Performance audit

```bash
systemctl status ozon-performance.service --no-pager
journalctl -u ozon-performance.service -n 100 --no-pager
systemctl cat ozon-performance.service
git -C /opt/kvantexpert/marketplaces-mcp-ru rev-parse HEAD || true
```

Ожидаемый runtime:
`/opt/kvantexpert/marketplaces-mcp-ru/.venv/bin/ozon-perf-mcp`
Ожидаемый pinned commit: `ec2114595695536e001e09e1144a357118852db1`.

## Performance smoke

После проверки loopback:
1. local MCP initialize;
2. tools/list;
3. ozon_perf_describe_method;
4. harmless READ `ozonperf_get_api_client_campaign`.

Для первичного аудита не выполнять write/delete.

## 48-operation audit

Источник operation_id: `patches/marketplaces-mcp-ru/perf_endpoints.yaml`.
Для каждого operation_id вызвать `ozon_perf_describe_method`.
Ожидание: 48/48, 0 FAIL.

## 293 audit

Первым делом сделать свежий READ:
- product list;
- product limits;
- category tree.

Проверить `disabled` у category 200001489 и type 971075562.

Если disabled=true или import снова даёт used_forbidden_category: остановить импорт.

Если категория разрешена: только тогда готовить первую партию <=100 и проверять task/status.

## Security / stop conditions

Остановиться, если:
- :8001 опубликован наружу;
- service не active;
- nginx -t fails;
- env permissions != 600;
- catalog != 48;
- category remains disabled;
- появляются неожиданные изменения в runtime.

Не записывать в документацию Client-Id, API-Key, Performance secret, TLS private key или digital codes.