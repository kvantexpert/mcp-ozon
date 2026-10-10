# Ozon MCP — настройка, deployment и recovery

Дата: 2026-09-27

## 1. Что развернуто

Один VPS:
- Seller MCP :8000 loopback
- Performance MCP :8001 loopback
- nginx :80/:443
- Seller public endpoint: https://ozon-mcp.kvantexpert.ru/mcp
- Performance public endpoint отсутствует.

Seller:
- ozon-mcp-ru 0.6.0
- uvx runtime
- env /root/.config/ozon-mcp/env

Performance:
- marketplaces-mcp-ru 0.6.1
- pinned commit ec2114595695536e001e09e1144a357118852db1
- venv /opt/kvantexpert/marketplaces-mcp-ru/.venv
- env /root/.config/ozon-mcp/perf.env

## 2. Проверка текущего сервера

```bash
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

Не выводить env contents.

Ожидается:
- services active/enabled;
- 8000/8001 loopback;
- nginx -t successful;
- env mode 600.

## 3. Seller deployment

Фактический production unit использует:

```text
/root/.local/bin/uvx --from 'ozon-mcp-ru==0.6.0' ozon-mcp-ru
```

Для диагностики:
```bash
systemctl cat ozon-mcp.service
systemctl restart ozon-mcp.service
systemctl is-active ozon-mcp.service
journalctl -u ozon-mcp.service -n 100 --no-pager
```

После restart:
1. local MCP initialize;
2. tools/list;
3. harmless READ;
4. public endpoint smoke.

## 4. Performance deployment

Runtime source-of-truth:
- upstream commit ec2114595695536e001e09e1144a357118852db1;
- tracked patched catalog patches/marketplaces-mcp-ru/perf_endpoints.yaml;
- installer deploy/scripts/install-performance-mcp.sh;
- unit deploy/systemd/ozon-performance.service.

Повторный deployment:
1. checkout repo;
2. run installer;
3. create perf.env on server;
4. daemon-reload;
5. enable/start service;
6. check 127.0.0.1:8001;
7. initialize/tools/list;
8. 48/48 describe;
9. harmless READ.

Seller :8000 не менять.

## 5. Nginx

Public only Seller.

Не добавлять Performance route.

Проверка:
```bash
nginx -t
systemctl reload nginx
```

Если public Seller не отвечает:
1. nginx status;
2. nginx logs;
3. Seller status;
4. Seller local endpoint;
5. only then inspect Ozon API.

## 6. Credentials

Seller:
`/root/.config/ozon-mcp/env`

Performance:
`/root/.config/ozon-mcp/perf.env`

Mode:
`600 root:root`

Никогда не коммитить:
- Client-Id;
- Api-Key;
- Performance client secret;
- Bearer token;
- TLS private key;
- cookies;
- digital codes.

## 7. Seller MCP smoke

Local initialize:
```bash
curl -sS -D /tmp/ozon-mcp-headers.txt -o /tmp/ozon-mcp-init.json   -X POST 'http://127.0.0.1:8000/mcp'   -H 'Content-Type: application/json'   -H 'Accept: application/json, text/event-stream'   --data-binary '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"ozon-audit","version":"1.0"}}}'
grep -i '^mcp-session-id:' /tmp/ozon-mcp-headers.txt
cat /tmp/ozon-mcp-init.json
```

После restart session ID нужно получать заново.

## 8. Performance smoke

Порядок:
1. initialize;
2. tools/list;
3. harmless read:
`ozon_perf_call_method(operation_id=ozonperf_get_api_client_campaign)`;
4. затем при необходимости key READ operations.

Не выполнять write/delete в базовом audit.

## 9. 48 operation audit

Взять operation_id из:
`patches/marketplaces-mcp-ru/perf_endpoints.yaml`

Для каждого:
`ozon_perf_describe_method`

Ожидание:
48/48, 0 FAIL.

Это catalog audit, а не business success audit.

## 10. 293 import recovery

Каждый раз перед batch:

1. fresh product list;
2. limits;
3. category tree;
4. check disabled;
5. only then prepare <=100 items.

Если:
- `description_category_is_empty` -> category missing in payload;
- `used_forbidden_category` -> category unavailable, stop import.

После разрешённой category:
1. batch 100;
2. task_id;
3. import/info;
4. log exact errors;
5. batch 100;
6. batch 93.

## 11. Ozon API changes

Если endpoint returns 404/410 или wrapper кажется устаревшим:

search_methods
-> describe_method
-> inspect path/schema
-> READ test
-> update docs/code.

Не возвращаться автоматически к старому endpoint.

## 12. Full recovery checklist

```text
[ ] Git HEAD known
[ ] services active
[ ] 8000 loopback
[ ] 8001 loopback
[ ] nginx valid
[ ] env permissions 600
[ ] Seller MCP initialize
[ ] Performance MCP initialize
[ ] Performance 48/48 describe
[ ] harmless Performance READ
[ ] current category tree
[ ] 293 blocker status known
[ ] no secrets exposed
```


## 13. GitHub Deploy Key сервера ozon-mcp — read-only

Назначение: дать отдельному системному пользователю `ozon-desktop` на VPS `ozon-mcp` возможность читать репозиторий `kvantexpert/mcp-ozon` по SSH (например, `git ls-remote`, `clone` и `fetch/pull`) без персонального GitHub token. Этот ключ не предназначен для записи в GitHub и не даёт доступ к другим репозиториям.

### Единое название и расположение

- **GitHub Deploy Key title:** `ozon-mcp — kvantexpert/mcp-ozon — read-only`
- **Файл приватного ключа на VPS:** `/home/ozon-desktop/.ssh/github-mcp-ozon-readonly`
- **Файл публичного ключа:** `/home/ozon-desktop/.ssh/github-mcp-ozon-readonly.pub`
- **Комментарий ключа:** `github-mcp-ozon-readonly@ozon-mcp`
- **Алгоритм:** Ed25519
- **Fingerprint:** `SHA256:hDympwkG/dda3OuIspC95922NmhNQkQQEPwOWGrcPwA`
- **Дата генерации:** 2026-10-10

Имя состоит из роли/сервера, репозитория и уровня доступа. Такое имя использовать в GitHub, в имени файлов и в документации, чтобы позднее однозначно определить назначение ключа.

### Регистрация в GitHub

1. Открыть репозиторий https://github.com/kvantexpert/mcp-ozon/settings/keys.
2. Нажать **Add deploy key**.
3. В поле **Title** указать точно: `ozon-mcp — kvantexpert/mcp-ozon — read-only`.
4. В поле **Key** вставить публичный ключ из файла `/home/ozon-desktop/.ssh/github-mcp-ozon-readonly.pub`.
5. Оставить **Allow write access** выключенным. Нужен только READ-доступ.
6. После сохранения проверить SSH-доступ командой `git ls-remote` к `git@github.com:kvantexpert/mcp-ozon.git` от пользователя `ozon-desktop`.

### Безопасность и текущий статус

- Приватный ключ остаётся на VPS в `/home/ozon-desktop/.ssh/` с правами доступа только владельцу; его запрещено отправлять в чат, добавлять в репозиторий или копировать в документацию.
- В GitHub добавляется только публичный ключ (.pub).
- Deploy Key привязан только к одному репозиторию. Для другого репозитория создаётся отдельный ключ.
- **Статус на 2026-10-10:** публичный ключ добавлен в GitHub Deploy keys; SSH-аутентификация подтверждена с VPS командой `git ls-remote git@github.com:kvantexpert/mcp-ozon.git HEAD`. Проверенный ответ HEAD: `d5469e154fd03a555fa7569b89ff6f6b99c82dd8`.
- Для подключения настроен `/home/ozon-desktop/.ssh/config`: хост `github.com`, `IdentityFile=/home/ozon-desktop/.ssh/github-mcp-ozon-readonly`, `IdentitiesOnly=yes`, `StrictHostKeyChecking=yes`.
- GitHub host key проверен по ED25519 fingerprint `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`; доверенный ключ хранится в `/home/ozon-desktop/.ssh/known_hosts`.
- Успешно выполнена только READ-проверка; WRITE-команды не запускались. В настройках Deploy Key параметр **Allow write access** должен оставаться выключенным.
- Файлы на сервере: приватный ключ — режим `600`, публичный ключ — режим `644`.
- Создание ключа и подготовка документации не требуют рестарта `ozon-mcp.service` или `ozon-performance.service`; сервисы не перезапускать ради настройки Deploy Key.
