# Ozon MCP — аудит production VPS

Дата актуализации: 2026-09-27.

Важно: этот документ описывает безопасную процедуру аудита. В текущем сообщении VPS напрямую не опрашивался.

## 1. Host/runtime

Ожидаемая среда:
- host cv7976275
- Ubuntu 22.04.4 LTS
- Seller :8000
- Performance :8001
- nginx :80/:443

## 2. Safe audit

```bash
set -e

echo '=== HOST ==='
hostname
hostname -I
lsb_release -ds 2>/dev/null || true

echo '=== SERVICES ==='
systemctl is-enabled ozon-mcp.service
systemctl is-active ozon-mcp.service
systemctl show ozon-mcp.service -p ActiveState -p SubState -p MainPID -p ExecMainStartTimestamp

systemctl is-enabled ozon-performance.service
systemctl is-active ozon-performance.service
systemctl show ozon-performance.service -p ActiveState -p SubState -p MainPID -p ExecMainStartTimestamp

echo '=== PORTS ==='
ss -lntp | grep -E ':8000|:8001' || true

echo '=== NGINX ==='
nginx -t

echo '=== SECRET PERMISSIONS ==='
stat -c '%a %U:%G %n'   /root/.config/ozon-mcp/env   /root/.config/ozon-mcp/perf.env
```

Не выводить env contents.

## 3. Seller audit

```bash
systemctl status ozon-mcp.service --no-pager
journalctl -u ozon-mcp.service -n 100 --no-pager
systemctl cat ozon-mcp.service
```

Expected launch:
`/root/.local/bin/uvx --from 'ozon-mcp-ru==0.6.0' ozon-mcp-ru`

## 4. Performance audit

```bash
systemctl status ozon-performance.service --no-pager
journalctl -u ozon-performance.service -n 100 --no-pager
systemctl cat ozon-performance.service
git -C /opt/kvantexpert/marketplaces-mcp-ru rev-parse HEAD || true
ss -lntp | grep ':8001' || true
```

Expected upstream:
`ec2114595695536e001e09e1144a357118852db1`

## 5. MCP smoke

Seller:
- local initialize;
- tools/list;
- harmless READ.

Performance:
- local initialize;
- tools/list;
- `ozon_perf_call_method(operation_id=ozonperf_get_api_client_campaign)`.

Do not execute write/delete during baseline audit.

## 6. 48-operation audit

Source:
`patches/marketplaces-mcp-ru/perf_endpoints.yaml`

Run `ozon_perf_describe_method` for all operation_id.

Expected:
- 48/48;
- 0 FAIL.

This is catalog audit, not business success audit.

## 7. 293 audit

Always start with fresh READ:
- product list;
- product limits;
- category tree.

For target category/type:
- 200001489;
- 971075562.

If `disabled=true`:
**stop import**.

If import returns:
- `description_category_is_empty` -> payload category missing;
- `used_forbidden_category` -> category unavailable.

## 8. Security stop conditions

Stop and fix infrastructure before functional work if:
- 8001 is externally bound;
- either service inactive;
- nginx -t fails;
- env permissions not 600;
- unexpected runtime catalog;
- unexpected uncommitted changes;
- credentials appear in logs/output.

## 9. Current documentation status

Last documented Performance live catalog audit:
26.09.2026 -> 48/48, 0 FAIL.

Last documented 293 negative tests:
24.09.2026 -> category disabled/forbidden, import not started.

A fresh server audit is the first step before the next functional deployment.
