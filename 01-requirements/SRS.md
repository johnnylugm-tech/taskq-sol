# Software Requirements Specification (SRS) — `taskq-api`

## 1. Introduction

### 1.1 Purpose

`taskq-api` 將任務佇列服務化：用戶可透過 REST API 提交、查詢與執行任務；資料持久化於關聯式資料庫；schema 可隨版本演進；服務支援認證、授權與流量控制。

**Canonical citation:** `SPEC.md:49-54`.

### 1.2 Product form

- 語言：Python 3.11。
- 形態：ASGI 服務，以 `uvicorn taskq_api.app:app` 啟動。
- 管理入口：`python -m taskq_api`，提供 `migrate`、`seed`、`healthcheck`。
- 開發與測試資料庫為 SQLite；生產資料庫為 PostgreSQL；兩者使用同一份 ORM 模型。

**Canonical citation:** `SPEC.md:52-54`, `SPEC.md:64`.

### 1.3 Scope and source of truth

本 SRS 轉錄 canonical `SPEC.md` v1.0.0 所宣告的 10 條 FR、12 條 NFR、HTTP/CLI 介面、資料與設定邊界。若本 SRS 的解釋與 canonical 規格衝突，以 `SPEC.md` 為準；未能由 canonical 文字唯一決定的項目列於 §7。

**Canonical citation:** `SPEC.md:16-23`, `SPEC.md:458`.

## 2. Constraints

### 2.1 Technical architecture

| Concern | Required technology or boundary |
|---|---|
| HTTP | FastAPI（ASGI） |
| Validation | Pydantic v2 request/response models |
| ORM | SQLAlchemy 2.x declarative ORM，使用 `Session` 明確劃分交易邊界 |
| Database | SQLite（開發/測試）與 PostgreSQL（生產），共用 ORM 模型 |
| Migration | Alembic，v1 → v2 → v3，每步皆有 `downgrade` |
| Async | `async def` endpoints 與 `asyncio.TaskGroup` 背景執行器 |
| Authentication | `X-API-Key`；只儲存金鑰雜湊 |
| Authorization | Per-token `read` / `write` / `admin` scope |
| Rate limiting | Per-token token bucket |
| Error media type | RFC 7807 `application/problem+json` |
| Task execution | `asyncio.create_subprocess_exec`；禁止 `shell=True` |
| Layering | `import-linter` layers contract |

**Canonical citation:** `SPEC.md:58-72`.

### 2.2 Environment configuration

`config.py` 讀取下列 12 個變數，`.env.example` 必須逐一宣告並附註解：

| Variable | Default | Canonical meaning |
|---|---:|---|
| `TASKQ_DB_URL` | `sqlite:///./taskq.db` | 資料庫連線字串；不得出現在日誌 |
| `TASKQ_DB_POOL_SIZE` | `5` | 連線池大小 |
| `TASKQ_TASK_TIMEOUT` | `10.0` | 單任務 subprocess timeout（秒） |
| `TASKQ_MAX_CONCURRENT` | `8` | 背景執行併發上限 |
| `TASKQ_DRAIN_TIMEOUT` | `30.0` | 關閉時 graceful drain 上限（秒） |
| `TASKQ_RATE_BURST` | `20` | 令牌桶容量 |
| `TASKQ_RATE_PER_SEC` | `5.0` | 令牌補充速率 |
| `TASKQ_CORS_ORIGINS` | 空字串 | 逗號分隔的允許來源；空值代表全部拒絕 |
| `TASKQ_LOG_LEVEL` | `INFO` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `TASKQ_LOG_FORMAT` | `json` | `json` / `text` |
| `TASKQ_HOST` | `127.0.0.1` | 監聽位址；預設不對外 |
| `TASKQ_PORT` | `8000` | 監聽埠 |

**Canonical citation:** `SPEC.md:285-302`, `SPEC.md:381`.

### 2.3 Canonical database schema

| Table | Revision | Required principal fields |
|---|---|---|
| `tasks` | v1 | `id`（uuid）、`command`、`name`、`status`、`created_at`；`result_json` 在 v1 建立並於 v3 移除 |
| `api_keys` | v1 | `id`、`key_hash`（sha256）、`scope`、`created_at`、`revoked_at` |
| `tags` | v2 | `id`、`label` |
| `task_tags` | v2 | `task_id`、`tag_id`（複合主鍵） |
| `task_results` | v3 | `id`、`task_id`（FK）、`exit_code`、`stdout_tail`、`stderr_tail`、`duration_ms`、`finished_at` |
| `rate_buckets` | v1 | `key_id`（FK）、`tokens`、`updated_at` |

`tasks.result_json` 的既有資料在 v3 搬遷至 `task_results`；往返 migration 必須驗證資料可逆。v1 revision 的表集合在 canonical 文字中有衝突，保留於 §7 的 `NFR-99`，不得由本 SRS 自行選定解釋。

**Canonical citation:** `SPEC.md:133-142`, `SPEC.md:303-315`.

### 2.4 Required project files

| File | Required purpose |
|---|---|
| `.importlinter` | Layer contract 與 `sqlalchemy` forbidden contract |
| `requirements.txt` + `requirements.lock` | Direct pinning 與 transitive locking |
| `requirements-dev.txt` | `import-linter`、`pip-licenses`、`mutmut`、`pytest-benchmark`、`httpx` |
| `alembic.ini` + `migrations/versions/` | FR-07 的三個 revision |
| `.env.example` | §2.2 的 12 個 `TASKQ_*` 變數 |
| `.methodology/harness_config.json` | `features.mutation_testing: true`；不得調降 `crg_cohesion_healthy` |
| `Makefile` | `verify-system`，包含 migration 往返 |
| `08-config/SBOM.json` | 完整依賴樹 SBOM |

**Canonical citation:** `SPEC.md:316-326`, `SPEC.md:239`, `SPEC.md:424`.

### 2.5 Framework-aligned verification constraints

- Canonical framework dimensions另要求 `linting`、`type_safety`、`test_coverage`、`architecture` 與 `secrets_scanning` 的既定 gate；整體 source line coverage 門檻為 100%。
- `crg_cohesion_healthy` 保持預設值，不得為通過而調降。
- `taskq_api.service.runner`、`taskq_api.service.auth`、`taskq_api.repository.session`、`migrations/versions/v3_split_results.py` 為高風險模組，需 per-module TDD 覆蓋。
- 若 `ast-error-handling` 或 `ast-assertions` 對 async 語法誤判或漏判，須在 Phase 4 bug hunt 記錄，不得靜默繞過。

**Canonical citation:** `SPEC.md:405-428`, `SPEC.md:441`.

## 3. Functional Requirements

### FR-01: 任務資源 CRUD API

提供具 scope 授權、驗證、cursor pagination 與一致錯誤契約的任務 CRUD API。

**Acceptance criteria**

#### AC-1.1

`POST /v1/tasks` 要求 `write` scope；body 由 `TaskCreate` pydantic 模型驗證；有效請求回傳 HTTP 201 與 task id。

**Canonical citation:** `SPEC.md:82`, `SPEC.md:359`.

#### AC-1.2

DERIVED: SPEC.md line 83 — Canonical endpoint table row transcribed as prose without changing behavior.

`GET /v1/tasks/{id}` 要求 `read` scope，並取得單一任務全欄位。

**Canonical citation:** `SPEC.md:83`.

#### AC-1.3

DERIVED: SPEC.md line 84 — Canonical endpoint table row transcribed as prose without changing query parameters.

`GET /v1/tasks` 要求 `read` scope，提供分頁列表並支援 `?status=`、`?limit=`、`?cursor=`。

**Canonical citation:** `SPEC.md:84`.

#### AC-1.4

分頁為 cursor-based，不得使用 offset；列表端點預設 `limit` 為 50、上限為 200，超過上限回傳 HTTP 422。

**Canonical citation:** `SPEC.md:89-90`.

#### AC-1.5

DERIVED: SPEC.md line 85 — Canonical endpoint table row transcribed as prose without changing scope or transaction boundary.

`DELETE /v1/tasks/{id}` 要求 `admin` scope；刪除任務與結果列必須在同一交易完成。

**Canonical citation:** `SPEC.md:85`.

#### AC-1.6

`TaskCreate` request validation 失敗時回傳 HTTP 422 與 `application/problem+json`。

**Canonical citation:** `SPEC.md:87`, `SPEC.md:332-337`.

#### AC-1.7

未知 task id 回傳 HTTP 404、`application/problem+json`，且 `type` 為 `/errors/not-found`。

**Canonical citation:** `SPEC.md:88`, `SPEC.md:339`, `SPEC.md:362`.

#### AC-1.8

DERIVED: SPEC.md lines 87-87; lines 340-340 — Name uniqueness is paired with the canonical 409 conflict mapping.

`POST /v1/tasks` 使用重複 name 時回傳 HTTP 409，且 `type` 為 `/errors/conflict`。

**Canonical citation:** `SPEC.md:87`, `SPEC.md:340`, `SPEC.md:363`.

### FR-02: 任務執行端點

DERIVED: SPEC.md lines 92-99 — Canonical bullets are consolidated into a requirement summary; criteria retain each behavior.

提供非同步任務啟動與歷史執行結果查詢。

**Acceptance criteria**

#### AC-2.1

`POST /v1/tasks/{id}/run` 要求 `write` scope，回傳 HTTP 202 Accepted，body 含 `run_id`。

**Canonical citation:** `SPEC.md:94`.

#### AC-2.2

DERIVED: SPEC.md lines 95-95; lines 292-292 — Canonical execution and timeout clauses are paired with the canonical default value.

實際執行使用 `asyncio.create_subprocess_exec(*shlex.split(command))`，禁止 `shell=True`；timeout 為 `TASKQ_TASK_TIMEOUT`，預設 `10.0` 秒。

**Canonical citation:** `SPEC.md:95`, `SPEC.md:292`.

#### AC-2.3

DERIVED: SPEC.md line 96 — Canonical state-machine notation is transcribed as a standalone criterion.

任務狀態機為 `pending → running → done | failed | timeout`。

**Canonical citation:** `SPEC.md:96`.

#### AC-2.4

執行結果寫入 FR-07 v3 schema 的 `task_results` 表，包含 `exit_code`、`stdout_tail`、`stderr_tail`、`duration_ms`、`finished_at`。

**Canonical citation:** `SPEC.md:97`, `SPEC.md:311`.

#### AC-2.5

DERIVED: SPEC.md line 98 — Canonical history endpoint bullet is transcribed as prose without changing ordering.

`GET /v1/tasks/{id}/runs` 要求 `read` scope，回傳該任務的歷史執行紀錄，排序由新到舊。

**Canonical citation:** `SPEC.md:98`.

### FR-03: API Key 認證

DERIVED: SPEC.md lines 100-107 — Canonical authentication bullets are consolidated into a requirement summary; criteria retain each behavior.

全部受保護 API 使用 `X-API-Key`，且不持久化明文金鑰。

**Acceptance criteria**

#### AC-3.1

全部 `/v1/*` 端點要求 `X-API-Key` header；缺少或無效時回傳 HTTP 401、`application/problem+json`，且 `type` 為 `/errors/unauthenticated`。

**Canonical citation:** `SPEC.md:102`, `SPEC.md:337`, `SPEC.md:360`.

#### AC-3.2

DERIVED: SPEC.md lines 103-103; lines 373-373 — Canonical storage and comparison clauses are paired with their canonical table-verification expectation.

金鑰以 SHA-256 雜湊儲存於 `api_keys` 表，不得存明文；比對使用 `hmac.compare_digest`。查表時 `key_hash` 為 64 hex 且無明文金鑰。

**Canonical citation:** `SPEC.md:103`, `SPEC.md:308`, `SPEC.md:373`.

#### AC-3.3

`python -m taskq_api key create --scope <scope>` 產生金鑰，明文只在建立當下印出一次。

**Canonical citation:** `SPEC.md:104`.

#### AC-3.4

`revoked_at` 非空的金鑰一律視為無效。

**Canonical citation:** `SPEC.md:105`.

#### AC-3.5

DERIVED: SPEC.md line 106 — Canonical unauthenticated endpoint exception is transcribed as a standalone criterion.

`/healthz` 與 `/readyz` 不要求認證。

**Canonical citation:** `SPEC.md:106`, `SPEC.md:155-156`.

### FR-04: Scope 授權

以單一 dependency 執行階層式 scope 授權。

**Acceptance criteria**

#### AC-4.1

DERIVED: SPEC.md line 110 — Canonical scope hierarchy is restated in prose without changing inclusion semantics.

每把金鑰帶一個 scope，階層為 `read` < `write` < `admin`，高階 scope 包含低階權限。

**Canonical citation:** `SPEC.md:110`.

#### AC-4.2

端點所需 scope 依 FR-01/FR-02；scope 不足回傳 HTTP 403 與 `application/problem+json`，`type` 為 `/errors/forbidden`，body 不得洩漏該資源是否存在。

**Canonical citation:** `SPEC.md:111`, `SPEC.md:338`, `SPEC.md:361`.

#### AC-4.3

DERIVED: SPEC.md line 112 — Canonical single-dependency implementation boundary and named verification are kept together.

授權判定必須在單一中介層 dependency 完成，不得散落於 handlers；測試須斷言每個 `/v1` route 都經過同一個 dependency。

**Canonical citation:** `SPEC.md:112`.

### FR-05: 流量控制

對每個 token 使用持久化 token bucket，且 health endpoints 不受限。

**Acceptance criteria**

#### AC-5.1

DERIVED: SPEC.md lines 116-116; lines 295-296 — Canonical token-bucket variables are paired with defaults from the canonical configuration table.

Per-token 令牌桶容量為 `TASKQ_RATE_BURST`（預設 20），補充速率為 `TASKQ_RATE_PER_SEC`（預設 5.0）。

**Canonical citation:** `SPEC.md:116`, `SPEC.md:295-296`.

#### AC-5.2

DERIVED: SPEC.md lines 117-117; lines 341-341; lines 364-364 — Canonical rate-limit response is paired with its canonical trigger example and error type.

超限回傳 HTTP 429、`application/problem+json`、`type: /errors/rate-limited` 與以秒為單位的 `Retry-After` header；連續請求超過 `TASKQ_RATE_BURST` 可觸發該回應。

**Canonical citation:** `SPEC.md:117`, `SPEC.md:341`, `SPEC.md:364`.

#### AC-5.3

令牌桶狀態存於資料庫以維持跨 worker 一致；更新必須在單一交易內以 row-level lock 進行。

**Canonical citation:** `SPEC.md:118`.

#### AC-5.4

`/healthz` 與 `/readyz` 不受 rate limit。

**Canonical citation:** `SPEC.md:119`.

### FR-06: 持久化層與交易邊界

DERIVED: SPEC.md lines 121-128 — Canonical persistence bullets are consolidated into a requirement summary; criteria retain each boundary.

所有資料操作遵循 repository boundary 與 request-scoped transaction。

**Acceptance criteria**

#### AC-6.1

全部資料存取經由 `repository/` 層；業務層不得直接持有 `Session`。

**Canonical citation:** `SPEC.md:123`.

#### AC-6.2

每個 API request 使用一個 `Session`；成功 commit、例外 rollback，並以 context manager 保證交易邊界。

**Canonical citation:** `SPEC.md:124`.

#### AC-6.3

禁止字串拼接 SQL；一律使用 ORM 或參數化查詢。

**Canonical citation:** `SPEC.md:125`.

#### AC-6.4

DERIVED: SPEC.md line 126 — Canonical eager-loading requirement is transcribed as a standalone criterion.

關聯查詢必須以 `selectinload` 或 `joinedload` 顯式預載；N+1 是驗收失敗條件。

**Canonical citation:** `SPEC.md:126`.

#### AC-6.5

DERIVED: SPEC.md lines 127-127; lines 290-291 — Canonical pool settings are paired with defaults from the canonical configuration table.

連線池使用 `pool_size=TASKQ_DB_POOL_SIZE`（預設 5）及 `pool_pre_ping=True`；`TASKQ_DB_URL` 預設為 `sqlite:///./taskq.db`。

**Canonical citation:** `SPEC.md:127`, `SPEC.md:290-291`.

### FR-07: Schema Migration（Alembic 三步演進）

Alembic schema 以三個可逆 revision 演進，v3 包含真實資料搬遷。

**Acceptance criteria**

#### AC-7.1

存在 v1、v2、v3 三個 revision，且每一步皆有可運作的 `downgrade`。

**Canonical citation:** `SPEC.md:131-137`.

#### AC-7.2

DERIVED: SPEC.md lines 135-135; lines 312-312 — Canonical v1 row is retained verbatim in substance while its internal conflict is deferred to NFR-99.

Canonical v1 revision 要求建立 `tasks`、`api_keys` 兩表；`downgrade` drop 兩表。

**Canonical citation:** `SPEC.md:135`, `SPEC.md:312`.

#### AC-7.3

v2 新增 `tags`、`task_tags`（多對多）及 `tasks.name` 唯一索引；`downgrade` drop 新表與索引且不影響 v1 資料。

**Canonical citation:** `SPEC.md:136`.

#### AC-7.4

v3 將 `tasks.result_json` 的既有資料搬遷至獨立 `task_results` 表後移除原欄位；`downgrade` 反向搬遷回 `tasks.result_json` 後 drop `task_results`，資料不得遺失。

**Canonical citation:** `SPEC.md:137`, `SPEC.md:314`.

#### AC-7.5

`alembic upgrade head` 與 `alembic downgrade base` 都必須成功；`alembic downgrade base` exit 0 且無殘留表。

**Canonical citation:** `SPEC.md:139`, `SPEC.md:368`.

#### AC-7.6

往返可逆性驗收為 `upgrade head` → 寫入樣本資料 → `downgrade -1` → `upgrade head`，樣本資料欄位值逐欄相同。

**Canonical citation:** `SPEC.md:140`, `SPEC.md:367`.

#### AC-7.7

禁止以 `op.execute("DROP TABLE ...")` 等破壞性捷徑取代真正的 `downgrade`。

**Canonical citation:** `SPEC.md:141`.

#### AC-7.8

Migration 檔本身納入測試覆蓋，以 Alembic offline SQL 產生並斷言。

**Canonical citation:** `SPEC.md:142`.

#### AC-7.9

DERIVED: SPEC.md lines 303-314 — Canonical schema table rows are referenced as one head-schema criterion without adding fields.

Head schema 包含 §2.3 所列 `tasks`、`api_keys`、`tags`、`task_tags`、`task_results`、`rate_buckets` 及其 canonical principal fields。

**Canonical citation:** `SPEC.md:303-314`.

### FR-08: 非同步執行器

以有界、可 drain、可取消且不遺留子進程的 async runner 執行任務。

**Acceptance criteria**

#### AC-8.1

背景執行以 `asyncio.TaskGroup` 管理；服務關閉時 graceful drain，等待進行中的任務至 `TASKQ_DRAIN_TIMEOUT`（預設 30.0 秒），逾時標記 `interrupted`。

**Canonical citation:** `SPEC.md:146`, `SPEC.md:294`, `SPEC.md:380`.

#### AC-8.2

併發上限為 `TASKQ_MAX_CONCURRENT`（預設 8）；超過時新任務排隊，不得無限制生成 coroutine。

**Canonical citation:** `SPEC.md:147`, `SPEC.md:293`.

#### AC-8.3

DERIVED: SPEC.md lines 148-148 — Canonical timeout cleanup clause is paired with its canonical orphan-process acceptance target.

任務 timeout 以 `asyncio.wait_for` 實作；逾時須 `process.kill()` 後 `await process.wait()`，不得留下孤兒進程。

**Canonical citation:** `SPEC.md:148`, `SPEC.md:397`, `SPEC.md:452`.

#### AC-8.4

`asyncio.CancelledError` 必須向上傳播，不得被 `except Exception` 吞掉，亦不得轉成 HTTP 500。

**Canonical citation:** `SPEC.md:149`, `SPEC.md:346`.

### FR-09: 健康檢查與可觀測性

DERIVED: SPEC.md lines 151-160 — Canonical endpoint table is consolidated into a requirement summary; criteria retain endpoint behavior.

提供 unauthenticated liveness/readiness 與 admin metrics。

**Acceptance criteria**

#### AC-9.1

DERIVED: SPEC.md line 155 — Canonical health endpoint table row is transcribed as prose without changing response content.

`GET /healthz` 不需認證；進程存活時回傳 HTTP 200 與 `{"status":"ok"}`。

**Canonical citation:** `SPEC.md:155`.

#### AC-9.2

`GET /readyz` 不需認證；DB 連線可用且 `alembic current == head` 時回傳 HTTP 200；任一不成立時 fail closed，回傳 HTTP 503、`type: /errors/not-ready`，body detail 指明失敗項目。

**Canonical citation:** `SPEC.md:156`, `SPEC.md:159`, `SPEC.md:342`, `SPEC.md:365-366`.

#### AC-9.3

`GET /v1/metrics` 要求 `admin` scope，回傳按狀態分組的任務計數、執行延遲分位數與 rate-limit 拒絕數。

**Canonical citation:** `SPEC.md:158`.

### FR-10: 錯誤契約（RFC 7807）

所有非 2xx response 使用一致、可關聯且不洩漏內部資訊的 problem details contract。

**Acceptance criteria**

#### AC-10.1

全部非 2xx 回應的 `Content-Type` 為 `application/problem+json`。

**Canonical citation:** `SPEC.md:163`, `SPEC.md:332`.

#### AC-10.2

Problem body 欄位為 `type`（URI）、`title`、`status`、`detail`、`instance`、`correlation_id`。

**Canonical citation:** `SPEC.md:164`.

#### AC-10.3

`detail` 不得含 SQL 陳述、堆疊追蹤、檔案路徑或資料庫結構描述。

**Canonical citation:** `SPEC.md:165`, `SPEC.md:374`.

#### AC-10.4

同一 `correlation_id` 同時出現在 response header `X-Correlation-Id` 與伺服器日誌。

**Canonical citation:** `SPEC.md:166`.

#### AC-10.5

Request body 驗證失敗回傳 HTTP 422，`type` 為 `/errors/validation`。

**Canonical citation:** `SPEC.md:167`, `SPEC.md:336`.

#### AC-10.6

缺少或無效 API key 回傳 HTTP 401，`type` 為 `/errors/unauthenticated`。

**Canonical citation:** `SPEC.md:167`, `SPEC.md:337`.

#### AC-10.7

Scope 不足回傳 HTTP 403，`type` 為 `/errors/forbidden`，且不洩漏資源是否存在。

**Canonical citation:** `SPEC.md:167`, `SPEC.md:338`.

#### AC-10.8

DERIVED: SPEC.md lines 167-167; lines 339-339 — Canonical status mapping is paired with the canonical error type.

未知 task id 回傳 HTTP 404，`type` 為 `/errors/not-found`。

**Canonical citation:** `SPEC.md:167`, `SPEC.md:339`.

#### AC-10.9

DERIVED: SPEC.md lines 167-167; lines 340-340 — Canonical status mapping is paired with the canonical error type.

任務名稱衝突回傳 HTTP 409，`type` 為 `/errors/conflict`。

**Canonical citation:** `SPEC.md:167`, `SPEC.md:340`.

#### AC-10.10

超過 rate limit 回傳 HTTP 429，`type` 為 `/errors/rate-limited`，並附 `Retry-After`。

**Canonical citation:** `SPEC.md:167`, `SPEC.md:341`.

#### AC-10.11

DB 不可用或 migration 未到 head 時回傳 HTTP 503，`type` 為 `/errors/not-ready`。

**Canonical citation:** `SPEC.md:167`, `SPEC.md:342`.

#### AC-10.12

其他未預期例外回傳 HTTP 500，`type` 為 `/errors/internal`，且 detail 不含堆疊、SQL 或路徑。

**Canonical citation:** `SPEC.md:167`, `SPEC.md:344`.

#### AC-10.13

任務 timeout 是 HTTP 200 response 中的任務狀態 `timeout`，不使用 problem `type`。

**Canonical citation:** `SPEC.md:343`.

## 4. Non-Functional Requirements

### NFR-01: 效能與查詢效率

DERIVED: NFR-01 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `performance`

**Acceptance criteria**

#### AC-N1.1

DERIVED: SPEC.md lines 179-182 — Canonical latency target and named measurement method are combined without changing the threshold.

`GET /v1/tasks/{id}` 在 10,000 筆資料下 p95 < 30ms（不含網路，以 ASGI transport 量測），量測方式為 `pytest-benchmark`。

**Canonical citation:** `SPEC.md:179`, `SPEC.md:182`, `SPEC.md:370`, `SPEC.md:436`.

**Coverage note:** 現行 `performance` dimension 讀取 pytest-benchmark 的 mean 並使用 1000ms/3000ms 扣分門檻，未驗證本 AC 的 p95 < 30ms；需 dedicated benchmark assertion。

#### AC-N1.2

DERIVED: SPEC.md lines 180-182 — Canonical latency target and named measurement method are combined without changing the threshold.

`GET /v1/tasks?limit=50` 在 10,000 筆資料下 p95 < 80ms，量測方式為 `pytest-benchmark`。

**Canonical citation:** `SPEC.md:180`, `SPEC.md:182`, `SPEC.md:437`.

**Coverage note:** 現行 `performance` dimension 量測 mean，未驗證本 AC 的 p95 < 80ms；需 dedicated benchmark assertion。

#### AC-N1.3

列表端點一次 request 發出的 SQL 陳述數必須是常數、與回傳筆數無關；以 SQLAlchemy event listener 計數斷言，N+1 為失敗條件。

**Canonical citation:** `SPEC.md:181`, `SPEC.md:369`, `SPEC.md:438`.

**Coverage note:** 現行 `performance` dimension 不計 SQL statements，也不驗證 query-count invariance；需 dedicated N+1 test。

### NFR-02: HTTP 與資料層安全

DERIVED: NFR-02 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `security`

**Acceptance criteria**

#### AC-N2.1

DERIVED: SPEC.md lines 187-187; lines 371-371 — Canonical prohibition is paired with the exact verifier named by the canonical acceptance table.

全 codebase 禁用 `shell=True`、`eval(`、`exec(`，grep 0 命中；由 canonical 命令 `grep -rn "shell=True\|eval(\|exec(" 03-development/src/` 判定。

**Canonical citation:** `SPEC.md:187`, `SPEC.md:371`.

**Coverage note:** 現行 `security` dimension 執行 Bandit；不等同 canonical grep 的零命中規則，且不解決 §7 所列「全 codebase」與 `src/` 命令範圍差異。

#### AC-N2.2

禁止以 f-string、`%` 或 `+` 組成 SQL；一律使用 ORM 或參數化查詢，並以 grep 加 code review 雙重驗證，命中數為 0。

**Canonical citation:** `SPEC.md:188`, `SPEC.md:372`, `SPEC.md:448`.

**Coverage note:** 現行 `security` dimension 的 Bandit 不保證偵測全部 SQL 字串拼接形態；需 canonical grep 與 review。

#### AC-N2.3

DERIVED: SPEC.md line 189 — Canonical API-key storage and comparison clauses are transcribed as one criterion.

API key 雜湊儲存，並使用 `hmac.compare_digest` 比對。

**Canonical citation:** `SPEC.md:189`.

**Coverage note:** 現行 `security` dimension 不驗證資料庫內容或 compare function 的 runtime 使用；需 dedicated storage/auth test。

#### AC-N2.4

HTTP 403 body 不得洩漏資源存在性。

**Canonical citation:** `SPEC.md:190`.

**Coverage note:** 現行 `security` dimension 不比較存在與不存在資源的 403 response；需 dedicated integration test。

#### AC-N2.5

Error body 不得含堆疊、SQL 或路徑。

**Canonical citation:** `SPEC.md:191`.

**Coverage note:** 現行 `security` dimension 不檢查 runtime error responses；需 dedicated integration test。

#### AC-N2.6

DERIVED: SPEC.md lines 192-192 — Canonical CORS rule is paired with the canonical default value.

CORS 預設拒絕所有來源；allowlist 由 `TASKQ_CORS_ORIGINS` 明示，預設空字串代表全拒。

**Canonical citation:** `SPEC.md:192`, `SPEC.md:297`.

**Coverage note:** 現行 `security` dimension 不發送 CORS requests；需 dedicated HTTP test。

#### AC-N2.7

DERIVED: SPEC.md line 193 — Canonical Bandit command and zero-severity target are transcribed as one criterion.

`bandit -r 03-development/src/` 的 HIGH 與 MEDIUM findings 均為 0。

**Canonical citation:** `SPEC.md:193`, `SPEC.md:378`, `SPEC.md:449`.

### NFR-03: 錯誤處理、交易與非同步正確性

DERIVED: NFR-03 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `error_handling`

**Acceptance criteria**

#### AC-N3.1

每個 request 的交易邊界明確：成功 commit、例外 rollback，並以 context manager 保證。

**Canonical citation:** `SPEC.md:198`.

**Coverage note:** 現行 `error_handling` dimension 計算有 handler 的 source files 比率，不驗證 transaction outcome；需 dedicated repository/integration tests。

#### AC-N3.2

DERIVED: SPEC.md line 199 — Canonical forbidden exception forms are transcribed as a standalone criterion.

不得出現裸 `except:` 或 `except Exception: pass`。

**Canonical citation:** `SPEC.md:199`.

**Coverage note:** 現行 `error_handling` dimension 只把未重新拋出的 bare `except:` 視為 anti-pattern；canonical 禁止任何 bare `except:`，故需 dedicated AST check。

#### AC-N3.3

`asyncio.CancelledError` 不得被吞掉，必須重新拋出。

**Canonical citation:** `SPEC.md:200`.

**Coverage note:** 現行 `error_handling` dimension 會標記部分 broad-handler anti-pattern，但不執行 cancellation propagation；需 dedicated async test。

#### AC-N3.4

DERIVED: SPEC.md line 201 — Canonical readiness failure and bounded-retry requirements are retained together.

資料庫連線失敗時 `/readyz` 回傳 HTTP 503 與明確 detail，不得靜默重試至無限。

**Canonical citation:** `SPEC.md:201`, `SPEC.md:365`.

**Coverage note:** 現行 `error_handling` dimension 不啟動 HTTP/DB failure path，也不計 retry 次數；需 dedicated integration test。

#### AC-N3.5

任務 timeout 必須確實終止子進程且不留下孤兒。

**Canonical citation:** `SPEC.md:202`.

**Coverage note:** 現行 `error_handling` dimension 不觀察 subprocess lifecycle；需 dedicated integration test。

#### AC-N3.6

Migration 失敗時交易 rollback，資料庫維持在前一個 revision。

**Canonical citation:** `SPEC.md:203`.

**Coverage note:** 現行 `error_handling` dimension 不執行 Alembic failure path；需 dedicated migration test。

### NFR-04: 敏感資料遮蔽

DERIVED: NFR-04 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `security`

**Acceptance criteria**

#### AC-N4.1

DERIVED: SPEC.md lines 208-209 — Canonical targets, regex and whole-line replacement behavior are transcribed as one criterion.

`stdout_tail`、`stderr_tail`、日誌與 error body 在落盤或送出前，凡匹配 `(sk-[A-Za-z0-9_-]{8,}|token=\S+|Bearer\s+\S+|postgres(ql)?://[^\s]+)` 的行，整行以 `[REDACTED]` 取代。

**Canonical citation:** `SPEC.md:208-209`.

**Coverage note:** 現行 `security` dimension 的 Bandit 不驗證 canonical regex 或 runtime redaction output；需 dedicated unit/integration tests。

#### AC-N4.2

DERIVED: SPEC.md lines 210-210; lines 375-375 — Canonical disclosure boundary is paired with its canonical acceptance scan target.

含密碼的資料庫連線字串不得出現在任何日誌、錯誤訊息或 `/v1/metrics` response；日誌與 metrics 全文不得含 `TASKQ_DB_URL` 的密碼片段。

**Canonical citation:** `SPEC.md:210`, `SPEC.md:375`, `SPEC.md:451`.

**Coverage note:** 現行 `security` dimension 不擷取日誌、error responses 或 metrics output；需 dedicated leak test。

#### AC-N4.3

API key 明文只在 `key create` 當下輸出一次，不得寫入任何持久化位置。

**Canonical citation:** `SPEC.md:211`.

**Coverage note:** 現行 `security` dimension 不觀察 CLI output 與 persistence；需 dedicated CLI/storage test。

### NFR-05: 文件覆蓋

DERIVED: NFR-05 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `documentation`

**Acceptance criteria**

#### AC-N5.1

全部公開函式與類別有 docstring，且 docstring 含 `[FR-XX]` 或 `[NFR-XX]` 引用；覆蓋率 100%。

**Canonical citation:** `SPEC.md:216`.

**Coverage note:** 現行 `documentation` dimension 驗證公開 API 是否有 docstring，但不驗證 FR/NFR citation 內容；需 dedicated citation scan。

#### AC-N5.2

每個 API endpoint 在 `/openapi.json` 中有 `summary` 與 `description`，由測試斷言。

**Canonical citation:** `SPEC.md:217`.

**Coverage note:** 現行 `documentation` dimension 不讀取 FastAPI OpenAPI schema；需 dedicated OpenAPI test。

### NFR-06: 架構分層契約

DERIVED: NFR-06 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `architecture_constraints`

**Acceptance criteria**

#### AC-N6.1

專案根目錄存在 `.importlinter` 並宣告 `api > service > repository > models` layers contract；上層可 import 下層，下層不得 import 上層；`config` 與 `errors` 為 independence 模組。

**Canonical citation:** `SPEC.md:222-228`.

**Coverage note:** 現行 `architecture_constraints` dimension 執行已宣告 contracts，但不獨立驗證設定檔是否精確宣告 canonical layers 與 independence modules；需 config-contract test。

#### AC-N6.2

Forbidden contract 要求 `repository` 以外的任何層不得 import `sqlalchemy`。

**Canonical citation:** `SPEC.md:229`.

**Coverage note:** 現行 dimension 會執行該 contract（若已正確宣告），但不證明 contract 本身未被縮窄；需檢查 `.importlinter` canonical content。

#### AC-N6.3

`lint-imports` exit 0，且 `service`/`api` 層 import `sqlalchemy` 會被阻擋。

**Canonical citation:** `SPEC.md:230`, `SPEC.md:376`, `SPEC.md:445-446`.

**Coverage note:** 現行 dimension 驗證目前的 `lint-imports` exit code；「加入 `service`/`api` 對 `sqlalchemy` 的 import 會被阻擋」仍取決於 canonical forbidden contract 是否精確存在，需 configuration-contract test。

#### AC-N6.4

DERIVED: SPEC.md line 231 — Canonical anti-weakening prohibitions are transcribed as a standalone criterion.

不得以刪除 `.importlinter`、萬用字元 `ignore_imports` 或降級 contract 取得通過。

**Canonical citation:** `SPEC.md:231`.

**Coverage note:** 現行 dimension 能將缺少 `.importlinter` 判為 unscoreable，但不比較 contract 強度或禁止 wildcard ignore；需 dedicated configuration test/review。

### NFR-07: 依賴與授權合規

DERIVED: NFR-07 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `license_compliance`

**Acceptance criteria**

#### AC-N7.1

全部 runtime direct dependencies 在 `requirements.txt` 以 `==` 釘版，transitive dependencies 以 `requirements.lock` 完整鎖定。

**Canonical citation:** `SPEC.md:236`.

**Coverage note:** 現行 `license_compliance` dimension 只以 ScanCode 掃描 source licenses，不驗證 requirement pinning 或 lock completeness；需 dedicated dependency-lock test。

#### AC-N7.2

完整依賴樹只允許 MIT、BSD-2-Clause、BSD-3-Clause、Apache-2.0、PSF license；其他 license 的 dependency 不得使用。

**Canonical citation:** `SPEC.md:237`, `SPEC.md:377`, `SPEC.md:447`.

**Coverage note:** 現行 dimension 的 ScanCode source scan 不建立 Python dependency tree，也不套用此 dependency allowlist；需 `pip-licenses` evidence 與 dedicated assertion。

#### AC-N7.3

DERIVED: SPEC.md line 238 — Canonical complete-tree scope and evidence command are transcribed as one criterion.

License 掃描範圍包含完整依賴樹（direct + transitive），證據命令為 `pip-licenses --format=json --with-system`。

**Canonical citation:** `SPEC.md:238`, `SPEC.md:377`.

**Coverage note:** 現行 dimension 執行 `scancode --license ... src/`，範圍窄於本 AC；本 AC 必須另行執行 canonical `pip-licenses` 命令。

#### AC-N7.4

產出 `08-config/SBOM.json`，每個 dependency 含 `name`、`version`、`license`、`direct|transitive`。

**Canonical citation:** `SPEC.md:239`.

**Coverage note:** 現行 dimension 不產生或驗證 SBOM artifact；需 dedicated artifact test。

### NFR-08: 變異測試

DERIVED: NFR-08 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `mutation_testing`

**Acceptance criteria**

#### AC-N8.1

DERIVED: SPEC.md line 244 — Canonical configuration path and feature flag are transcribed as a standalone criterion.

`.methodology/harness_config.json` 設定 `features.mutation_testing: true`。

**Canonical citation:** `SPEC.md:244`, `SPEC.md:325`.

**Coverage note:** 現行 `mutation_testing` dimension 量測 mutation result，不單獨驗證此 feature flag 的值；需 configuration assertion。

#### AC-N8.2

`mutmut` mutation score ≥ 70。

**Canonical citation:** `SPEC.md:245`, `SPEC.md:379`, `SPEC.md:444`.

#### AC-N8.3

DERIVED: SPEC.md line 246 — Canonical mutation scope and required rationale are transcribed as one criterion.

Mutation scope 限定於 `service/` 與 `repository/`，並在 `harness_config.json` 註記限定理由為執行時間預算。

**Canonical citation:** `SPEC.md:246`.

**Coverage note:** 現行 dimension 會計算所配置範圍的分數，但不驗證範圍恰為這兩層或理由文字；需 configuration assertion。

### NFR-09: 驗證真實性（零 skip 鐵律）

DERIVED: NFR-09 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `test_assertion_quality`

**Acceptance criteria**

#### AC-N9.1

DERIVED: SPEC.md line 251 — Canonical forbidden test forms are transcribed as a standalone criterion.

任何 FR/NFR 驗證測試不得使用 `pytest.skip`、`skipif`、`xfail` 或無斷言 stub。

**Canonical citation:** `SPEC.md:251`.

**Coverage note:** 現行 `test_assertion_quality` dimension 偵測 zero-assert tests，但不掃描 skip/skipif/xfail markers；需 dedicated collection/static check。

#### AC-N9.2

DERIVED: SPEC.md lines 252-252; lines 356-356 — Canonical test command and zero-skipped target are paired with the canonical all-green expectation.

`pytest 03-development/tests -q` 全部通過且 skipped 計數為 0。

**Canonical citation:** `SPEC.md:252`, `SPEC.md:356`, `SPEC.md:439`.

**Coverage note:** 現行 dimension 不以 pytest summary 的 skipped count 評分；需 canonical suite-run assertion。

#### AC-N9.3

DERIVED: SPEC.md line 253 — Canonical per-test assertion and zero-assert count are transcribed as one criterion.

每個 test function 至少有一個 `assert`，`zero_assert == 0`。

**Canonical citation:** `SPEC.md:253`, `SPEC.md:440`.

**Coverage note:** 現行 `test_assertion_quality` dimension 亦把 `self.assertXxx` 與 `pytest.raises` 視為 substantive assertion；canonical 文字要求至少一個 `assert`，故需 dedicated exact-policy check。

#### AC-N9.4

DERIVED: SPEC.md line 254 — Canonical anti-exclusion forms are transcribed as a standalone criterion.

不得以 `--ignore`、`-k`、`--deselect`、`collect_ignore` 或從 `testpaths` 移除目錄排除測試。

**Canonical citation:** `SPEC.md:254`.

**Coverage note:** 現行 dimension 只分析已收集/存在的 test functions，不偵測 collection exclusion；需 configuration/command review。

#### AC-N9.5

FR-07 三步 migration 必須用真實 SQLite file（非 in-memory mock）測試；往返可逆性以實際資料逐欄比對，不得降級為 skip。

**Canonical citation:** `SPEC.md:255`, `SPEC.md:443`.

**Coverage note:** 現行 dimension 不識別 database medium，也不執行 migration round trip；需 dedicated migration integration test。

#### AC-N9.6

DERIVED: SPEC.md line 256 — Canonical VERIFIED precondition is transcribed as a standalone traceability criterion.

`TRACEABILITY_MATRIX.md` 的 `VERIFIED` 只能在對應測試實際執行並通過時給出。

**Canonical citation:** `SPEC.md:256`.

**Coverage note:** 現行 dimension 不讀取 traceability status 與 test execution evidence 的一致性；需 traceability verification。

#### AC-N9.7

DERIVED: SPEC.md line 357 — Canonical coverage command and TOTAL target are transcribed as a standalone criterion.

`pytest 03-development/tests --cov=03-development/src --cov-report=term` 的 TOTAL source line coverage 為 100%。

**Canonical citation:** `SPEC.md:357`, `SPEC.md:441`.

**Coverage note:** 本 NFR 的 `test_assertion_quality` dimension 不量測 line coverage；此 AC 需由現行獨立 `test_coverage` dimension 與 canonical coverage command 驗證。

### NFR-10: 整合覆蓋

DERIVED: NFR-10 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `integration_coverage`

**Acceptance criteria**

#### AC-N10.1

DERIVED: SPEC.md line 261 — Canonical integration-suite path and line-coverage threshold are transcribed as one criterion.

只執行 `03-development/tests/integration/` 時，對 `03-development/src` 的 TOTAL line coverage ≥ 80%。

**Canonical citation:** `SPEC.md:261`, `SPEC.md:358`, `SPEC.md:442`.

#### AC-N10.2

DERIVED: SPEC.md line 262 — Canonical HTTP transport and forbidden direct-call boundary are transcribed together.

Integration tests 使用 `httpx.AsyncClient(transport=ASGITransport(app))` 驅動，不得直接呼叫 handler function。

**Canonical citation:** `SPEC.md:262`.

**Coverage note:** 現行 `integration_coverage` dimension 只量測 source line coverage，不檢查 client/transport 使用方式；需 dedicated static/runtime assertion。

#### AC-N10.3

Integration suite 至少涵蓋 CRUD 全鏈、401/403/404/409/422/429/503 各一例、migration 往返、rate-limit 觸發與恢復、graceful drain。

**Canonical citation:** `SPEC.md:263`.

**Coverage note:** 現行 dimension 不以 scenario inventory 評分；≥80% coverage 不保證本 AC 的各情境存在，需 dedicated tests 與 traceability。

### NFR-11: 可讀性

DERIVED: NFR-11 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `readability`

**Acceptance criteria**

#### AC-N11.1

DERIVED: SPEC.md line 268 — Canonical LLOC-weighted MI threshold is transcribed as a standalone criterion.

專案 MI（LLOC 加權）≥ 80。

**Canonical citation:** `SPEC.md:268`, `SPEC.md:453`.

**Coverage note:** 現行 `readability` dimension 計算所有檔案 MI 的簡單平均，不是 canonical 的 LLOC 加權 project MI；需 dedicated weighted-MI calculation。

#### AC-N11.2

DERIVED: SPEC.md line 268 — Canonical per-function CC threshold is split into its own stable criterion.

單一 function cyclomatic complexity ≤ 10。

**Canonical citation:** `SPEC.md:268`.

**Coverage note:** 現行 `readability` dimension 只讀取 radon MI，不驗證 per-function CC；需 dedicated complexity check。

#### AC-N11.3

DERIVED: SPEC.md line 269 — Canonical file and directory size limits are transcribed together.

單一 file ≤ 400 行，單一 directory ≤ 15 files。

**Canonical citation:** `SPEC.md:269`.

**Coverage note:** 現行 `readability` dimension 不驗證 file/directory size limits；需 dedicated filesystem check。

#### AC-N11.4

DERIVED: SPEC.md line 270 — Canonical handler-size and service-layer boundary are transcribed together.

每個 API handler ≤ 40 行，業務邏輯必須下沉至 `service/`。

**Canonical citation:** `SPEC.md:270`.

**Coverage note:** 現行 `readability` dimension 不辨識 handlers 或 service-layer placement；需 dedicated AST/architecture check。

### NFR-12: 系統驗證目標

DERIVED: NFR-12 — Canonical NFR is decomposed into stable criteria; its dimension is cross-checked against the current roster.

**dimension:** `execute_verification_target`

**Acceptance criteria**

#### AC-N12.1

DERIVED: SPEC.md lines 275-279 — Canonical ordered target steps are transcribed as one criterion.

`Makefile` 的 `verify-system` target 依序串接：`alembic upgrade head`、全套測試、服務啟動及 `/healthz`/`/readyz` smoke、`alembic downgrade base` 後再 `upgrade head`。

**Canonical citation:** `SPEC.md:275-279`, `SPEC.md:326`.

**Coverage note:** 現行 `execute_verification_target` dimension 只執行 target 並依 exit code 評分，不檢查 target 是否包含 canonical 四步；需 dedicated Makefile/system evidence assertion。

#### AC-N12.2

DERIVED: SPEC.md line 280 — Canonical exit status and stdout marker are transcribed together.

`make verify-system` exit 0，且 stdout 印出 `verify-system: PASS`。

**Canonical citation:** `SPEC.md:280`, `SPEC.md:382`, `SPEC.md:454`.

**Coverage note:** 現行 dimension 直接驗證 exit 0，但未驗證 stdout marker；marker 需 dedicated assertion。

#### AC-N12.3

DERIVED: SPEC.md lines 286-302; lines 381-381 — Canonical environment-count command is paired with the canonical 12-variable declaration table.

`.env.example` 逐一宣告 §2.2 的 12 個 `TASKQ_*` 變數並附註解；`grep -c "^TASKQ_" .env.example` 的結果為 12。

**Canonical citation:** `SPEC.md:286-302`, `SPEC.md:324`, `SPEC.md:381`.

**Coverage note:** 現行 dimension 不讀取 `.env.example`；需 dedicated configuration test。

#### AC-N12.4

`python -m taskq_api` 管理入口提供 `migrate`、`seed`、`healthcheck`。

**Canonical citation:** `SPEC.md:53`.

**Coverage note:** 現行 dimension 只保證 project-defined target exit 0；除非 target 明確涵蓋三個管理命令，仍需 dedicated CLI tests。

#### AC-N12.5

DERIVED: SPEC.md lines 53-53; lines 298-301 — Canonical ASGI command and configuration defaults/value sets are grouped as startup verification.

ASGI service 可由 `uvicorn taskq_api.app:app` 啟動；`TASKQ_HOST` 預設 `127.0.0.1`、`TASKQ_PORT` 預設 `8000`、`TASKQ_LOG_LEVEL` 預設 `INFO` 且可設為 `DEBUG` / `INFO` / `WARNING` / `ERROR`、`TASKQ_LOG_FORMAT` 預設 `json` 且可設為 `json` / `text`。

**Canonical citation:** `SPEC.md:53`, `SPEC.md:298-301`.

**Coverage note:** 現行 dimension 不獨立驗證 defaults/value boundaries；需 dedicated configuration/startup tests。

## 5. Acceptance Criteria Summary

| Requirement | Stable acceptance IDs | Primary canonical verifier |
|---|---|---|
| FR-01 | AC-1.1–AC-1.8 | HTTP integration tests for CRUD, validation, pagination and 404/409 |
| FR-02 | AC-2.1–AC-2.5 | Async runner and run-history integration tests |
| FR-03 | AC-3.1–AC-3.5 | Authentication, CLI and persisted-key tests |
| FR-04 | AC-4.1–AC-4.3 | Scope matrix and common-dependency tests |
| FR-05 | AC-5.1–AC-5.4 | Rate-limit trigger/recovery and transaction tests |
| FR-06 | AC-6.1–AC-6.5 | Repository, transaction, query and pool tests |
| FR-07 | AC-7.1–AC-7.9 | Real SQLite migration/offline SQL/round-trip tests |
| FR-08 | AC-8.1–AC-8.4 | Concurrency, drain, timeout, cancellation and orphan-process tests |
| FR-09 | AC-9.1–AC-9.3 | Health/readiness/metrics HTTP tests |
| FR-10 | AC-10.1–AC-10.13 | RFC 7807 response matrix tests |
| NFR-01 | AC-N1.1–AC-N1.3 | `pytest-benchmark` plus SQLAlchemy event listener |
| NFR-02 | AC-N2.1–AC-N2.7 | grep, code review, HTTP security tests and Bandit |
| NFR-03 | AC-N3.1–AC-N3.6 | Transaction, failure-path, async and migration tests |
| NFR-04 | AC-N4.1–AC-N4.3 | Redaction, log/metrics leak and persistence tests |
| NFR-05 | AC-N5.1–AC-N5.2 | AST citation scan and `/openapi.json` tests |
| NFR-06 | AC-N6.1–AC-N6.4 | `.importlinter` content checks and `lint-imports` |
| NFR-07 | AC-N7.1–AC-N7.4 | lock checks, `pip-licenses`, allowlist assertion and SBOM validation |
| NFR-08 | AC-N8.1–AC-N8.3 | harness config checks and framework mutation score |
| NFR-09 | AC-N9.1–AC-N9.7 | pytest summary, assertion scan, real migration and coverage evidence |
| NFR-10 | AC-N10.1–AC-N10.3 | Integration-only coverage and scenario traceability |
| NFR-11 | AC-N11.1–AC-N11.4 | Weighted MI, CC, size and handler-boundary checks |
| NFR-12 | AC-N12.1–AC-N12.5 | `make verify-system`, config, CLI and startup tests |

## 6. Out-of-Scope

Canonical `SPEC.md` 未宣告額外的 out-of-scope 功能。§1–§5 所轉錄的 endpoints、CLI、databases、migrations、configuration、quality thresholds 與 artifacts 均不得被視為範圍外；任何新增功能則需先變更 canonical 規格。

**Canonical citation:** `SPEC.md:2`, `SPEC.md:22-23`.

## 7. Open Issues

- **FR-01-deferred / NFR-99:** Resolve `SPEC.md:87` ambiguity in FR-01 — current SPEC 以「驗證規則同第 1 輪」引用 canonical source 以外的規格，括號雖列出「非空 / ≤1000 字元 / 注入字元黑名單 / 名稱唯一」，但未把前 3 條映射到具體 `TaskCreate` fields，也未列 blacklist 字元集合；可能解釋為只套用於 command，或分別套用於 command/name；test harness to confirm with stakeholder。未解決前不得自行新增 field mapping 或 blacklist。
- **FR-02-deferred / NFR-99:** Resolve `SPEC.md:97` ambiguity in FR-02 — current SPEC 命名 `stdout_tail` / `stderr_tail`，但未指定 tail 長度或截斷單位；可能解釋為固定字元數，或固定 bytes/lines；test harness to confirm with stakeholder。
- **FR-08-deferred / NFR-99:** Resolve the `SPEC.md:97,147` conflict between FR-02 and FR-08 — current SPEC defines the state machine as `pending → running → done | failed | timeout`, while graceful-drain timeout marks a task `interrupted`; it is ambiguous whether `interrupted` is an additional terminal state with a transition from `running`, or a marker outside that state machine; test harness to confirm with stakeholder.
- **FR-05-deferred / NFR-99:** Resolve `SPEC.md:64,118` ambiguity in FR-05 — current SPEC 同時要求 SQLite 開發/測試與 row-level lock；可能解釋為只在 PostgreSQL 驗證 row-level locking，或在 SQLite 驗證等價 serialization；test harness to confirm with stakeholder。
- **FR-07-deferred / NFR-99:** Resolve `SPEC.md:135,312,314` ambiguity in FR-07 — current SPEC 的 v1 migration row 說建立 `tasks`、`api_keys`「兩表」，schema 又將 `rate_buckets` 與 `tasks.result_json` 歸於 v1；可能解釋為 v1 實際建立三表並含該欄位，或另有未列 revision；test harness to confirm with stakeholder。
- **FR-09-deferred / NFR-99:** Resolve `SPEC.md:157` ambiguity in FR-09 — current SPEC 要求「執行延遲分位數」但未命名 percentile set；可能解釋為只需 p95，或需多個 percentile；test harness to confirm with stakeholder。
- **NFR-02-deferred / NFR-99:** Resolve `SPEC.md:187,371` ambiguity in NFR-02 — current SPEC 說掃描「全 codebase」，canonical acceptance command 只掃描 `03-development/src/`；可能解釋為 source-only gate，或 repository-wide gate；test harness to confirm with stakeholder。
- Prompt-injection scan outcome: 未在 canonical `SPEC.md` 偵測到 prompt-injection pattern，故無 clause 因此 deferred。

## 8. Risks

| ID | Risk | Impact | Likelihood | Canonical mitigation |
|---|---|---|---|---|
| R1 | v3 資料搬遷遺失資料 | 高 | 中 | 真實 DB migration 往返並逐欄比對（FR-07 / AC-7.6） |
| R2 | SQL injection | 高 | 低 | 禁止字串拼接，使用 ORM/參數化並設 grep gate（NFR-02） |
| R3 | API key 洩漏 | 高 | 中 | 雜湊儲存、常數時間比對、明文只印一次（FR-03） |
| R4 | 403 洩漏資源存在性 | 中 | 中 | 在資源查詢前完成授權判定（FR-04） |
| R5 | N+1 查詢在大表上崩潰 | 高 | 高 | 顯式預載與 SQL count assertion（NFR-01） |
| R6 | Error body 洩漏內部結構 | 中 | 高 | RFC 7807 固定欄位與 detail allowlist（FR-10） |
| R7 | `CancelledError` 被吞，關閉卡死 | 中 | 中 | 明文禁令與 propagation test（NFR-03） |
| R8 | 任務 timeout 留下孤兒進程 | 中 | 中 | `kill()` 後 `await wait()`（FR-08） |
| R9 | 部署後忘記 migration | 高 | 中 | `/readyz` fail closed（FR-09） |
| R10 | 連線池耗盡 | 中 | 中 | `pool_pre_ping` 與併發上限（FR-06/FR-08） |
| R11 | Transitive dependency 引入不相容 license | 中 | 中 | Lock file 與完整依賴樹掃描（NFR-07） |
| R12 | Rate bucket race 導致超放行 | 低 | 中 | 單一 transaction 與 row-level lock（FR-05） |

**Canonical citation:** `SPEC.md:386-401`.

## 9. Glossary

| Term | Definition in this SRS |
|---|---|
| API key | 經 `X-API-Key` 傳入、以 SHA-256 hash 持久化的 credential |
| ASGI | 本服務採用的 Python async server interface；由 FastAPI/uvicorn 提供 |
| Cursor-based pagination | 使用 `cursor` 而非 SQL offset 的列表分頁方式 |
| Graceful drain | 關閉服務時等待進行中任務至 `TASKQ_DRAIN_TIMEOUT` 的行為 |
| N+1 | Query 數量隨回傳 records 增加的失敗型態；本規格要求 statement count 為常數 |
| Problem details | RFC 7807 `application/problem+json` error response |
| Scope | API key 的階層權限：`read` < `write` < `admin` |
| SBOM | `08-config/SBOM.json` 中列出 direct/transitive dependencies 的 artifact |
| Token bucket | 由容量與每秒速率控制、per-token 且持久化的 rate-limit state |
| Transaction boundary | 每 request 成功 commit、例外 rollback 的 SQLAlchemy `Session` lifecycle |
| v1/v2/v3 | FR-07 定義的三個 Alembic schema revisions |

## FR Block (machine-readable)

```json
{
  "functional_requirements": [
    {
      "id": "FR-01",
      "description": "具 scope 授權、validation、cursor pagination 與一致錯誤契約的任務 CRUD API。",
      "implementation_functions": ["FastAPI handlers for POST/GET/DELETE /v1/tasks"],
      "verification_method": "AC-1.1–AC-1.8 HTTP integration tests"
    },
    {
      "id": "FR-02",
      "description": "以 async subprocess 執行任務並查詢執行歷史。",
      "implementation_functions": ["taskq_api.service.runner", "POST /v1/tasks/{id}/run", "GET /v1/tasks/{id}/runs"],
      "verification_method": "AC-2.1–AC-2.5 runner and HTTP integration tests"
    },
    {
      "id": "FR-03",
      "description": "X-API-Key authentication、雜湊儲存與 key lifecycle。",
      "implementation_functions": ["taskq_api.service.auth", "python -m taskq_api key create"],
      "verification_method": "AC-3.1–AC-3.5 authentication, CLI and storage tests"
    },
    {
      "id": "FR-04",
      "description": "單一 dependency 實施 read/write/admin 階層式授權。",
      "implementation_functions": ["taskq_api.service.auth"],
      "verification_method": "AC-4.1–AC-4.3 scope matrix and route-dependency tests"
    },
    {
      "id": "FR-05",
      "description": "Database-backed per token token bucket rate limiting。",
      "implementation_functions": ["per-token token-bucket dependency", "rate_buckets"],
      "verification_method": "AC-5.1–AC-5.4 rate-limit and transaction tests"
    },
    {
      "id": "FR-06",
      "description": "Repository-only data access 與 request-scoped transaction boundaries。",
      "implementation_functions": ["repository/", "taskq_api.repository.session"],
      "verification_method": "AC-6.1–AC-6.5 architecture, transaction and query tests"
    },
    {
      "id": "FR-07",
      "description": "可 downgrade 的 Alembic v1/v2/v3 schema evolution 與 v3 data migration。",
      "implementation_functions": ["Alembic v1/v2/v3 revisions", "migrations/versions/v3_split_results.py"],
      "verification_method": "AC-7.1–AC-7.9 real-SQLite migration round-trip and offline SQL tests"
    },
    {
      "id": "FR-08",
      "description": "有界、可 graceful drain、正確取消且不留 orphan process 的 async runner。",
      "implementation_functions": ["taskq_api.service.runner"],
      "verification_method": "AC-8.1–AC-8.4 async lifecycle and subprocess tests"
    },
    {
      "id": "FR-09",
      "description": "Unauthenticated health/readiness endpoints 與 admin metrics。",
      "implementation_functions": ["taskq_api.app:app", "GET /healthz", "GET /readyz", "GET /v1/metrics"],
      "verification_method": "AC-9.1–AC-9.3 HTTP and database-readiness tests"
    },
    {
      "id": "FR-10",
      "description": "RFC 7807 error contract、correlation id 與內部細節抑制。",
      "implementation_functions": ["errors", "application/problem+json exception handlers"],
      "verification_method": "AC-10.1–AC-10.13 response-contract matrix tests"
    }
  ],
  "non_functional_requirements": [
    {
      "id": "NFR-01",
      "type": "performance",
      "description": "10k-row p95 latency thresholds and constant SQL statement count。",
      "test_method": "AC-N1.1–AC-N1.3 pytest-benchmark and SQLAlchemy event-listener assertions"
    },
    {
      "id": "NFR-02",
      "type": "security",
      "description": "HTTP、subprocess、SQL、API-key、CORS 與錯誤輸出 security boundaries。",
      "test_method": "AC-N2.1–AC-N2.7 grep, code review, integration tests and Bandit"
    },
    {
      "id": "NFR-03",
      "type": "reliability",
      "description": "Transaction rollback、async cancellation、subprocess cleanup 與 migration failure correctness。",
      "test_method": "AC-N3.1–AC-N3.6 failure-path and async integration tests"
    },
    {
      "id": "NFR-04",
      "type": "security",
      "description": "Canonical-pattern redaction and credential/database URI protection。",
      "test_method": "AC-N4.1–AC-N4.3 redaction, logging, metrics, CLI and persistence tests"
    },
    {
      "id": "NFR-05",
      "type": "documentation",
      "description": "100% cited public docstrings and OpenAPI summary/description coverage。",
      "test_method": "AC-N5.1–AC-N5.2 AST citation scan and OpenAPI schema tests"
    },
    {
      "id": "NFR-06",
      "type": "layering",
      "description": "Import-linter layers、independence modules 與 sqlalchemy forbidden contract。",
      "test_method": "AC-N6.1–AC-N6.4 configuration checks and lint-imports"
    },
    {
      "id": "NFR-07",
      "type": "licensing",
      "description": "Pinned complete dependency tree、license allowlist and SBOM。",
      "test_method": "AC-N7.1–AC-N7.4 lock validation, pip-licenses and SBOM checks"
    },
    {
      "id": "NFR-08",
      "type": "mutation",
      "description": "Mutation testing enabled for service/repository with score at least 70。",
      "test_method": "AC-N8.1–AC-N8.3 harness-config assertions and framework mutation score"
    },
    {
      "id": "NFR-09",
      "type": "verifiability",
      "description": "Zero skipped/xfail/stub tests、substantive assertions、real migration tests and truthful traceability。",
      "test_method": "AC-N9.1–AC-N9.7 pytest, AST, migration, traceability and coverage evidence"
    },
    {
      "id": "NFR-10",
      "type": "integration",
      "description": "At least 80% integration source coverage with required HTTP and lifecycle scenarios。",
      "test_method": "AC-N10.1–AC-N10.3 integration coverage and scenario traceability"
    },
    {
      "id": "NFR-11",
      "type": "maintainability",
      "description": "Weighted MI、cyclomatic complexity、file/directory size and handler size boundaries。",
      "test_method": "AC-N11.1–AC-N11.4 weighted MI, complexity, filesystem and AST checks"
    },
    {
      "id": "NFR-12",
      "type": "deployability",
      "description": "Executable system verification target、complete environment declaration、CLI and service startup。",
      "test_method": "AC-N12.1–AC-N12.5 system target, configuration, CLI and startup tests"
    }
  ]
}
```
