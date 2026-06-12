[📚 Docs](../../README.md) › [Guides](../README.md) › Tester / QA

# Hướng Dẫn Tester — Spec-Driven Dev

Tài liệu dành cho **QA / Tester** — cách kết nối với spec framework, workflow kiểm thử, và các tình huống thực tế.

## Mục Lục

| Trang | Nội dung |
|---|---|
| [Spec Manifest & Setup](spec-manifest.md) | Spec manifest là gì · setup tester agent · Living Docs panel |
| [Workflow](workflow.md) | Luồng làm việc cơ bản từ nhận task đến pass/fail |
| [Đọc Spec Chain](reading-specs.md) | Thứ tự đọc PRD → BDD → Tech Docs · ví dụ thực tế |
| [Tình huống thực tế](scenarios.md) | 6 scenario: feature mới, PRD thay đổi, multi-service, regression, ... |
| [Báo cáo bug](bug-reporting.md) | `/report-bug` · `/propose-scenario` · template báo cáo |
| [Checklist test pass](test-checklist.md) | Checklist trước khi báo "test pass" |

## Vai Trò Tester Trong Framework

```
PO/BA          Dev                    Tester
──────────     ──────────────────     ──────────────────────────
PRD            BDD (từ PRD)           /sync + /generate-spec-manifest
               Tech Docs              Đọc PRD + BDD + Tech Docs
               Code                   Viết test cases · Chạy test
   ▲              ▲                    /report-bug      (bug → spec-traced)
   │              │                    /propose-scenario (edge case → BDD draft)
   └──────────────┴───── feedback/ trong spec repo ◄────┘
        PO/Dev thấy qua /sync (📥) → fix / promote / update PRD
```

**Tester chịu trách nhiệm:**
- Đọc PRD để hiểu business requirement trước khi test
- Đọc BDD để biết chính xác scenarios cần cover
- Đọc Tech Docs để hiểu API contracts và data flow
- Viết test cases dựa trên BDD scenarios
- Khi bug xảy ra: `/report-bug` → tạo bug report có spec-context đầy đủ, phân loại layer
- Khi phát hiện edge case chưa có trong BDD: `/propose-scenario` → đề xuất cho PO/Dev duyệt

**Tester KHÔNG làm:**
- Sửa PRD / BDD / Tech Docs trực tiếp — đó là việc của PO và Dev
- Approve PRD — chỉ PO
- Generate code — chỉ Dev

> **Lưu ý về `/propose-scenario`:** lệnh này **không** sửa BDD. Nó chỉ ghi một bản *đề xuất* (draft Gherkin) vào vùng `feedback/bdd-proposals/` của spec repo dùng chung. PO/Dev review và đưa vào `.feature` chính thức.

## Commands Dành Cho Tester

| Command | Mục đích | Khi nào dùng |
|---|---|---|
| `/generate-spec-manifest` | Tạo index TICKET-ID → PRD/BDD/tech-doc | Trước mỗi sprint, sau khi pull specs mới |
| `/report-bug {UC-ID} {mô tả}` | Tạo bug report có spec-context + phân loại layer (read-only) | Khi test fail / phát hiện bug |
| `/propose-scenario {UC-ID} {mô tả}` | Đề xuất scenario BDD mới cho edge case chưa cover | Khi tìm thấy case ngoài BDD hiện có |

**QC Automation Pipeline (chính thức) — 6 lệnh `/qc-*`:**

| Command | Mục đích | Khi nào dùng |
|---|---|---|
| `/qc-analyze` | Phân tích spec chain (PRD + BDD + Tech Docs), xác định scope test | Bước 1, sau khi BDD đã approved |
| `/qc-plan` | Lập test plan: liệt kê scenarios, layer, ưu tiên | Bước 2 |
| `/qc-design-test` | Thiết kế test case chi tiết (Page Object, data, assertions) | Bước 3 |
| `/qc-review` | Review test design trước khi code automation | Bước 4 |
| `/qc-run-test` | Chạy automation test → ghi `qc_status` per scenario vào trace TSV | Bước 5 |
| `/qc-report` | Tổng hợp kết quả thành QC report | Bước 6 |

Đây là **bộ test chính thức (authoritative) của QC**, chạy ngay trong framework. Pipeline dùng stack module `qc-playwright` (Python + pytest-playwright + Page Object). Xem chi tiết flow tại **[QC Automation guide](../qc-automation.md)**.

> **Phân biệt với test commands của Dev:** `/dev-gen-test`, `/dev-run-test`, `/dev-smoke-test` là **dev tự kiểm tra code của mình** — phát tín hiệu `dev_selftest`, KHÔNG phải bộ test chính thức. Bộ test authoritative của QC là pipeline `/qc-*` (sinh ra `qc_status`). Hai luồng tách biệt.

---

*Xem thêm:* [Developer Guide](../developer/README.md) · [QC Automation Guide](../qc-automation.md) · [Operations › Bug Flow](../../04-operations/bug-flow.md) · [Reference › Commands](../../05-reference/commands.md)
