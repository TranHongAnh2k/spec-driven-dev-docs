[📚 Docs](../README.md) › [Reference](README.md) › Trace TSV Schema

# Trace TSV Schema

> Schema chuẩn của `.trace/{UC-ID}.tsv` — file trace state per use-case. Mỗi UC một file: 1 header row + 1 data row mỗi scenario. Đây là nguồn dữ liệu cho `/validate-traces` và panel Living Docs.

---

## Mục lục

- [Canonical TSV header](#canonical-tsv-header)
- [Column reference](#column-reference)
- [Two independent signals: dev_selftest vs qc_status](#two-independent-signals-dev_selftest-vs-qc_status)
- [Status computation (validate-traces)](#status-computation-validate-traces)
- [@trace tags across artifacts](#trace-tags-across-artifacts)
- [trace-report.json](#trace-reportjson)

---

## Canonical TSV header

Tab-separated, một header row + một data row mỗi scenario. Header chính xác (từ `generate-bdd.tmpl`):

```
sc_id	sc_title	spec_ver	gen_ver	implemented_by	test_count	test_classes	dev_selftest	dev_selftest_at	qc_status	qc_run_at	prd_version	bdd_version	tech_doc_revision	prd_status	uc_status	fe_phase	status	last_updated
```

19 cột, theo đúng thứ tự trên.

---

## Column reference

| # | Column | Ý nghĩa | Giá trị khởi tạo (`/generate-bdd`) | Ai ghi |
|---|--------|---------|------------------------------------|--------|
| 1 | `sc_id` | `{UC-ID}-SC{N}` — khóa chính của row | `{UC-ID}-SC{N}` | generate-bdd |
| 2 | `sc_title` | Scenario title text | từ `.feature` | generate-bdd |
| 3 | `spec_ver` | `@trace.sc_version` hiện tại của scenario | từ `.feature` | generate-bdd / validate-traces (reconcile) |
| 4 | `gen_ver` | Version tại thời điểm code được sinh | `—` | generate-code |
| 5 | `implemented_by` | `ClassName.method` cài đặt scenario | `—` | generate-code |
| 6 | `test_count` | Số test verify scenario | `—` | dev-gen-test |
| 7 | `test_classes` | Danh sách test class | `—` | dev-gen-test |
| 8 | `dev_selftest` | Kết quả dev self-check: `pass` / `fail` / `not_run` | `—` | **dev-run-test** |
| 9 | `dev_selftest_at` | Ngày chạy dev self-check (`YYYY-MM-DD`) | `—` | **dev-run-test** |
| 10 | `qc_status` | Kết quả QC chính thức: `pass` / `fail` / `skip` / `not_run` | `—` | **qc-run-test** |
| 11 | `qc_run_at` | Ngày chạy QC (`YYYY-MM-DD`) | `—` | **qc-run-test** |
| 12 | `prd_version` | `@trace.prd_version` từ `.feature` header | từ `.feature` | generate-bdd |
| 13 | `bdd_version` | `@trace.bdd_version` từ `.feature` header | từ `.feature` | generate-bdd |
| 14 | `tech_doc_revision` | Tech-doc revision tại thời điểm codegen | `—` | generate-code |
| 15 | `prd_status` | `\| **Status** \|` từ PRD metadata | từ PRD | generate-bdd |
| 16 | `uc_status` | Trạng thái UC: `draft` / `approved` / … | `draft` (UC mới) | generate-bdd |
| 17 | `fe_phase` | FE phase khi implement (`ui` / `integration`) | `—` | generate-code `--phase` |
| 18 | `status` | Trạng thái tổng hợp: `OK` / `DRIFT` / `GAP` / `UNTRACKED` | `UNTRACKED` | validate-traces |
| 19 | `last_updated` | Ngày update gần nhất (`YYYY-MM-DD`) | today | mọi command ghi row |

> **Re-generation rules (generate-bdd):** SC đã có trong TSV & `spec_ver` không đổi → chỉ update `sc_title`, `prd_version`, `bdd_version`, `prd_status`, `uc_status`, `last_updated`. Nếu `spec_ver` đổi → thêm `spec_ver` và set `status = DRIFT` ngay. SC mới → append với `gen_ver`/`implemented_by`/`test_count`/`test_classes`/`dev_selftest`/`dev_selftest_at`/`qc_status`/`qc_run_at`/`tech_doc_revision` = `—`. SC bị xóa khỏi `.feature` → xóa row.

---

## Two independent signals: dev_selftest vs qc_status

Trace TSV mang **hai cột tín hiệu độc lập** — không bao giờ trộn:

| | `dev_selftest` (+ `dev_selftest_at`) | `qc_status` (+ `qc_run_at`) |
|---|--------------------------------------|------------------------------|
| Ghi bởi | `/dev-run-test` | `/qc-run-test` |
| Ý nghĩa | Dev tự smoke-check code mình vừa sinh | Kết quả QC automation **chính thức** |
| Giá trị | `pass` / `fail` / `not_run` | `pass` / `fail` / `skip` / `not_run` |
| Stack | Dev implementation module | `qc-playwright` module |
| Coverage chính thức? | **KHÔNG** — chỉ là dev self-check | **CÓ** — official QC result |
| Keyed by | `@trace.verifies={UC-ID}` | `@trace.verifies={UC-ID}-SC{N}` |

`/validate-traces` chỉ **đọc** hai cột này cho report — không bao giờ ghi đè (mỗi cột do command sở hữu nó quản lý). Living Docs hiển thị cả hai cạnh nhau.

---

## Status computation (validate-traces)

`/validate-traces` tính cột `status` theo thứ tự ưu tiên (rule đầu tiên khớp thì dừng):

| Rule | Status | Điều kiện |
|------|--------|-----------|
| 1 | `UNTRACKED` | `implemented_by == —` (chưa sinh code) |
| 2 | `GAP` | `implemented_by != —` AND (`test_count == —` OR `test_count == 0`) |
| 3 | `DRIFT` | `spec_ver != gen_ver` (scenario đổi sau lần codegen gần nhất) |
| 4 | `OK` | tất cả: `spec_ver == gen_ver`, `implemented_by != —`, `test_count > 0` |

Ngoài ra `/validate-traces` còn flag **`PRD_DRIFT`** (PRD version trong code/TSV trễ hơn PRD file hiện tại) và **`TECHDOC_DRIFT`** (code sinh từ tech-doc revision cũ).

---

## @trace tags across artifacts

Mọi artifact link với nhau qua `@trace.*` tags.

**`.feature` files:**
```gherkin
# @trace.id: FEAT-001-UC1
# @trace.service: web-admin
# @trace.module: react
# @trace.prd_version: 1.2
# @trace.bdd_version: 3
# @trace.api_source: existing   # brownfield: API đã tồn tại, contract lấy từ PRD
```
Per-scenario trong file: `# @trace.scenario: {UC-ID}-SC{N}`, `# @trace.sc_version: 1.0`, `# @trace.business_rules: ...`.

**Code:**
```java
// @trace.implements=FEAT-001-UC1-SC2
// @trace.prd_version=1.2
// @trace.bdd_version=3
// @trace.tech_doc_revision=2
```

**Dev self-check tests:**
```java
// @trace.verifies=FEAT-001-UC1
// @trace.service=api-backend
// @trace.test_type=integration
```

**QC tests** (`qc-playwright`, drive `qc_status`):
```python
# @trace.verifies={UC-ID}-SC{N}
# @trace.source=<official .feature path>
# @trace.test_type=functional|integration|e2e|non-functional
```

---

## trace-report.json

`/validate-traces` ghi `{paths.trace_dir}/trace-report.json` — single source of truth cho web dashboards. Cấu trúc rút gọn:

- `generated_at`, `domain`
- `summary` — aggregates: `total_prds`, `approved_prds`, `total_ucs`, `approved_ucs`, `draft_ucs`, `total_scs`, `coded_scs`, `tested_scs`, `code_coverage_pct`, `test_coverage_pct`, `drift_count`, `gap_count`, `untracked_count`, `dev_selftest_passing/failing/not_run`, `qc_passing/failing/skipped/not_run`, `tech_docs_count`
- `prds[]` → `ucs[]` → `scenarios[]` — mỗi scenario có đầy đủ các trường của TSV row (gồm `dev_selftest`, `dev_selftest_at`, `qc_status`, `qc_run_at`)
- `issues` — phân loại: `drift`, `gap`, `untracked`, `prd_version_drift`, `techdoc_drift`, mỗi entry kèm `fix` command gợi ý

**TSV `"—"` → JSON mapping:** `implemented_by: "—"` → `null` · `test_count: "—"` → `0` · `test_classes: "—"` → `[]` · `tech_doc_revision: "—"` → `0` · `dev_selftest: "—"` → `"not_run"` · `dev_selftest_at: "—"` → `null` · `qc_status: "—"` → `"not_run"` · `qc_run_at: "—"` → `null`.

> **Umbrella mode:** mỗi service submodule giữ `.trace/` riêng (committed). `/validate-traces` (hoặc `/sync`) publish merged report vào `{spec_source}/.living-docs/` (canonical) + mirror `./.trace/trace-report.json` ở workspace hiện tại. Cả hai là generated mirror — gitignore, không commit.

---

*Xem thêm:* [Command Reference](commands.md) (`/validate-traces`, `/qc-run-test`, `/dev-run-test`) · [03 · Concepts](../03-concepts/) (traceability system).
