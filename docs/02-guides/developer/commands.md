[📚 Docs](../../README.md) › [Guides](../README.md) › [Developer](README.md) › Commands

# Commands Dành Cho Dev

| Command | Mục đích | Khi nào dùng |
|---|---|---|
| `/sync` `[spec-branch]` | **One-command setup hoặc update** — git pull + submodule sync + Living Docs refresh. Truyền branch để override branch spec submodule (vd `/sync develop`) | **Mỗi sáng trước khi bắt đầu work** |
| `/update-framework` | Nâng cấp **bản thân framework** (`.agent/commands/`, steps/, modules/) từ npm | Khi có version framework mới — không đụng project-context/CLAUDE.md |
| `/review-context {prd-file}` | Đọc + xác nhận PRD + BDD đủ rõ trước khi code — fan-out review dimension thành sub-agent song song + completeness-critic loop, findings file đầy đủ ngay trong 1 lần chạy | **Bước đầu tiên** khi nhận PRD mới |
| `/generate-tech-docs {prd-file}` | Sinh Tech Docs (API, DB schema, sequence diagram) | Sau khi đọc BDD từ spec submodule |
| `/generate-code {bdd-file}` | Sinh code — BE hoặc FE khi API đã sẵn sàng | Sau khi tech docs `approved` |
| `/generate-code {bdd-file} --phase=ui` | FE: gen UI + mock adapter (BE chưa ready) | Ngay sau khi đọc BDD |
| `/generate-code {bdd-file} --phase=integration` | FE: wire API thật thay mock | Sau khi sign-off gate `approved` |
| `/dev-gen-test {bdd-file}` | **Dev self-check** — sinh test cases từ BDD để dev tự verify code mình vừa gen (KHÔNG phải bộ test chính thức của QC/dev-team) | Song song hoặc sau generate-code |
| `/review-code {file}` | Review code theo 4 lens (Security/Perf/Arch/Test) | Trước khi tạo PR |
| `/review-tech-docs {tech-doc-file}` | Review chất lượng Tech Docs | Sau generate-tech-docs |
| `/dev-run-test` | **Dev self-check** — chạy test do dev tự gen để xác nhận code mình hoạt động (smoke/self-verify, KHÔNG phải coverage chính thức) — *umbrella mode: tự `cd` vào service_root, dùng service's `test_command`*. Ghi `dev_selftest` (pass/fail) vào trace TSV | Sau khi code + tests sẵn sàng |
| `/fix-bug {issue}` | Phân tích + fix bug có trace | Khi có bug report |
| `/debug {symptom}` | Debug vấn đề chưa rõ nguyên nhân | Khi cần trace root cause |
| `/dev-smoke-test` | **Dev self-check** — kiểm tra nhanh các luồng chính của code mình vừa làm (smoke, không thay thế bộ test chính thức) | Sau deploy hoặc merge lớn |
| `/validate-traces` | Kiểm tra toàn bộ trace chain còn hợp lệ | Sau refactor hoặc khi PRD update |
| `/learn {text}` | Ghi lại lỗi AI hay lặp thành guardrail | Khi AI lặp lại lỗi mà bạn không muốn nó tái diễn |

> **Dev self-check vs QC chính thức:** `/dev-gen-test` · `/dev-run-test` · `/dev-smoke-test` (ghi `dev_selftest`) chỉ là **smoke self-check của riêng dev**. Bộ QC chính thức giờ là native pipeline `/qc-analyze → /qc-plan → /qc-design-test → /qc-review → /qc-run-test → /qc-report` — **do QC chạy, không phải việc của dev** — và ghi `qc_status` riêng. Chi tiết: [QC Automation Guide](../qc-automation.md).

> Danh mục đầy đủ mọi command: [Reference › Commands](../../05-reference/commands.md).

## Project Lessons — dạy framework không lặp lỗi

AI đôi khi lặp đi lặp lại một lỗi trong dự án (vd: gọi repository thẳng từ controller, quên null-check). Thay vì sửa thủ công mỗi lần, **ghi lại thành "lesson"** — context-loader sẽ nạp nó vào đầu **mọi** lệnh như một ràng buộc cứng.

**2 cách ghi nhận:**
```bash
# Cách 1 — chủ động
/learn AI hay gọi repository thẳng từ controller, phải đi qua service layer

# Cách 2 — tự động: khi /review-code, /fix-bug, /debug phát hiện lỗi lặp lại
# → nó hỏi "Record as a project lesson? (Y/N)" → Y
```

**Lưu ở đâu:** `paths.lessons_file` (mặc định `specs/domain-knowledge/lessons-learned.md`; umbrella: `.agent/project-lessons.md` mỗi service). **Commit file này** để cả team cùng được bảo vệ.

> Đây là **bộ nhớ dự án**, không phải fine-tune model — lesson được nạp vào context mỗi lần chạy, nên AI "nhớ" và không lặp lại. Xem `[CTX LOADED]` có dòng `Lessons: loaded — N guardrails`.

## Xử lý feedback từ tester

Tester gửi bug report (`/report-bug`) và đề xuất scenario (`/propose-scenario`) vào `feedback/` của **spec repo**. Khi dev chạy `/sync`, nó liệt kê:
```
📥 New tester feedback (pulled this sync):
   Bug reports:    BUG-20260608-01  FT-001 — ...  [layer: Code]
   Scenario proposals: FT-001-trailing-spaces → AC2 (pending review)
```

Dev hành động theo phân loại:
- **Bug report** → `/fix-bug {BUG-ID}` (report đã có sẵn spec-context + AC bị vi phạm + layer)
- **Scenario proposal map vào AC sẵn có** → thêm scenario vào `.feature` (hoặc `/generate-bdd` lại), rồi `/generate-code` + `/dev-gen-test`
- **Proposal là yêu cầu mới (PRD change request)** → chuyển PO sửa PRD trước

> Bug reports có thể đến từ hai nguồn: Tester dùng `/report-bug` trực tiếp, **hoặc** từ kết quả QC automation (`qc_status: fail` trong `.trace/*.tsv` → tester chạy `/report-bug` → `/sync` → dev thấy tại đây). Cả hai đều dùng cùng luồng `/fix-bug`.

> Tester chỉ *đề xuất* trong `feedback/` — dev/PO mới đưa vào BDD chính thức. Giữ đúng ownership.

## Khi nào dùng `--phase` cho FE/App?

| Tình huống | Command |
|---|---|
| API **đã có sẵn** và đang hoạt động | `/generate-code {file}` — không flag, gen real API ngay |
| BE **chưa ready**, FE muốn bắt đầu ngay | `/generate-code {file} --phase=ui` — UI + mock adapter |
| Sign-off gate xong, cần wire API thật | `/generate-code {file} --phase=integration` |
| BE implement (system BDD) | `/generate-code {file}` — không flag |

> `--phase` chỉ có giá trị khi BE chưa sẵn sàng. Nếu API đã live → bỏ qua `--phase`, chạy thẳng default.

---

← [Developer Guide](README.md) · Tiếp theo: [BDD & Trace System](bdd-and-trace.md)
