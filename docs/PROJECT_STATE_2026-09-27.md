# Ozon MCP — каноническое состояние проекта / точка продолжения

Дата актуализации: 2026-09-27

Перед любой следующей работой сначала читать этот файл и docs/KNOWLEDGE_BASE.md.

## 1. Repository

GitHub:
https://github.com/kvantexpert/ozon-mcp

Branch:
main

Current HEAD:
1ca9644e84b7fb992ebd072cebdc968c666e7ad2

Current commit:
docs: add AI agent context for seller and performance

Commit statuses:
statuses=[]; обязательного CI gate нет.

## 2. Architecture

Seller and Performance are separate services on one VPS.

Seller:
- 127.0.0.1:8000
- public https://ozon-mcp.kvantexpert.ru/mcp
- ozon-mcp-ru 0.6.0

Performance:
- 127.0.0.1:8001
- no public route
- marketplaces-mcp-ru 0.6.1
- pinned upstream ec2114595695536e001e09e1144a357118852db1
- patched 48-operation catalog

Do not merge these runtimes.

## 3. Server

- host cv7976275
- Ubuntu 22.04.4 LTS
- nginx 80/443
- Seller service ozon-mcp.service
- Performance service ozon-performance.service
- env /root/.config/ozon-mcp/env
- Performance env /root/.config/ozon-mcp/perf.env
- expected credentials permissions 600 root:root

Последняя документированная server check: 2026-09-26.

Важно: это не означает свежий live audit на текущую дату; перед изменением runtime выполнить server/SERVER_AUDIT_2026-09-27.md.

Перед следующей функциональной работой выполнить свежий server audit.

## 4. Seller — доказано

Эталонная digital card:
- offer_id 4601546116680
- product_id 6417979753
- sku 5865629857
- category 200001489
- type_id 971075562
- price 23000 RUB
- VAT 0

Проверено:
- create;
- read;
- characteristics write/read;
- dictionary lookup;
- image import + primary;
- moderation approved;
- validation success;
- digital stock 0 -> 1;
- independent stock READ.

Не доказано:
- real order;
- real posting;
- digital code upload for real posting;
- delivery.

## 5. 293

Точный набор:
- 00000002 = 193
- 00000003 = 100
- total 293

Batches:
100 + 100 + 93

Price rule:
00001 -> 00003 -> 00004

Endpoint:
POST /v3/product/import

Status:
**BLOCKED**

Last documented blocker:
- category/type returned disabled=true;
- import without category -> description_category_is_empty;
- import with 200001489 -> used_forbidden_category.

Do not retry mass import until fresh category tree shows a permitted category/type.

Important:
293 positions were NOT imported.

## 6. Performance — completed migration

Completed:
- pinned 0.6.1 runtime;
- tracked 48-op catalog;
- 3 additions;
- safety corrections;
- systemd deployment;
- loopback :8001;
- 48/48 live describe audit.

48/48 only proves catalog loading and operation visibility.

## 7. Immediate next steps

1. Fresh server audit.
2. Post-migration harmless Performance READ:
   `ozon_perf_call_method(operation_id=ozonperf_get_api_client_campaign)`.
3. Key Performance READ coverage.
4. Fresh category tree/limits/product list.
5. If category is now allowed, prepare first 100 of 293; otherwise stop.
6. After Performance READ coverage, design minimal AI-visible tool set.
7. Only after explicit access-control design consider public Performance endpoint.
8. Digital posting/code/delivery remains a separate later stage.

## 8. Canonical documents

- docs/KNOWLEDGE_BASE.md
- docs/SETUP_AND_RECOVERY.md
- server/SERVER_AUDIT_2026-09-27.md
- server/SERVER_STATE.md
- server/VPS_RUNTIME.md
- server/PERFORMANCE_RUNTIME.md
- docs/OZON_OPERATIONS.md
- docs/MASS_CATALOG_IMPORT_PLAN.md
- docs/PERFORMANCE_API_MATRIX_2026-09-26.md
- docs/PERFORMANCE_MCP_BASELINE_2026-09-26.md
- docs/TEST_HISTORY.md
- docs/AI_MAINTENANCE.md
