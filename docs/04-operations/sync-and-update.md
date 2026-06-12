[📚 Docs](../README.md) › [Operations](README.md) › Sync & Update

# Sync & Update — Vận hành hằng ngày

> Cách đồng bộ nội dung dự án (`/sync`), nâng cấp framework (`/update-framework`), làm việc ở umbrella mode với git submodule, đồng bộ Living Docs, và workflow hằng ngày theo từng role.

---

## Mục lục

1. [Hai loại "update" — đừng nhầm](#1-hai-loại-update--đừng-nhầm)
2. [`/sync` — đồng bộ nội dung dự án](#2-sync--đồng-bộ-nội-dung-dự-án)
3. [`/update-framework` — nâng cấp framework](#3-update-framework--nâng-cấp-framework)
4. [Umbrella mode & git submodule](#4-umbrella-mode--git-submodule)
5. [Living Docs sync](#5-living-docs-sync)
6. [Workflow hằng ngày theo role](#6-workflow-hằng-ngày-theo-role)
7. [Project Lessons — không để AI lặp lỗi](#7-project-lessons--không-để-ai-lặp-lỗi)
8. [Câu hỏi thường gặp](#8-câu-hỏi-thường-gặp)

> Pipeline QC native (`/qc-*`) có trang riêng — xem [Guide · QC Automation](../02-guides/qc-automation.md). Ở đây chỉ điểm qua chỗ `/sync` chạm vào QC.

---

## 1. Hai loại "update" — đừng nhầm

Có 2 thứ cần update, **nguồn khác nhau, tần suất khác nhau**:

| | `/sync` | `/update-framework` |
|---|---|---|
| **Update cái gì** | Nội dung dự án: code/specs trong submodule + Living Docs | Bản thân framework: `.agent/commands/`, steps/, modules/ |
| **Nguồn** | Git remote của các submodule | npm registry |
| **Tần suất** | Mỗi sáng / trước khi work | Khi có version framework mới (hiếm) |
| **Đụng `project-context.yaml` / `CLAUDE.md`?** | Không | Không (chỉ refresh file framework) |

```bash
# Update nội dung dự án — chạy thường xuyên
/sync

# Nâng cấp framework — chạy khi có bản mới
/update-framework
```

---

## 2. `/sync` — đồng bộ nội dung dự án

`/sync` gộp `git pull` + cập nhật submodule + refresh Living Docs vào **1 lệnh duy nhất**, chạy từ umbrella root:

- Pull specs (spec submodule) + services (service submodules).
- Refresh Living Docs panel (regenerate trace report — xem [§5](#5-living-docs-sync)).
- Liệt kê **📥 New tester feedback** — bug report / scenario proposal mới từ tester trong vùng `feedback/` của spec repo.

```bash
/sync
# hoặc override branch của spec submodule một lần:
/sync develop
```

**Branch nào được kéo?** `/sync` đọc `submodule.<spec>.branch` trong `.gitmodules`; nếu không khai báo → lấy **default branch của remote** (`origin/HEAD`, thường `main`). PO làm trên branch khác (vd `develop`) → pin một lần cho cả team:

```bash
git config -f .gitmodules submodule.my-project-specs.branch develop
git add .gitmodules && git commit -m "chore: pin spec submodule branch = develop"
```

Service submodules không bị ảnh hưởng — luôn checkout đúng SHA mà umbrella ghi, không phụ thuộc branch.

---

## 3. `/update-framework` — nâng cấp framework

```bash
/update-framework
# → đọc .agent/FRAMEWORK_VERSION, so với npm
# → chạy npx @latest --init
# → review `git diff .agent/` rồi commit
```

> **Umbrella mode:** `/update-framework` chỉ cần chạy ở **umbrella root**. Service submodule chỉ chứa `.agent/project-context.yaml` (config), không có command files → không cần nâng cấp riêng.

---

## 4. Umbrella mode & git submodule

Claude Code luôn mở ở **umbrella repo** (không mở trong service submodule). Umbrella thấy được cả spec submodule (read-only) lẫn service submodule (code).

### 4.1 — Clone lần đầu

```bash
git clone --recurse-submodules https://gitlab.company.com/my-project-web.git

# Hoặc nếu đã clone mà submodule trống:
git submodule update --init --recursive
```

### 4.2 — Kiểm tra trạng thái submodule

```bash
git submodule status
```

| Ký hiệu | Ý nghĩa | Cách xử lý |
|---|---|---|
| `abc1234 my-project-specs` | Bình thường | — |
| `+abc1234 …` | Umbrella có thay đổi chưa commit | `git add <sub> && git commit` |
| `-abc1234 …` | Chưa init | `git submodule update --init` |
| `Uabc1234 …` | Conflict | Giải quyết conflict rồi commit |

### 4.3 — Cập nhật spec submodule (lấy PRD mới từ PO)

```bash
# Cách 1: kéo version mới nhất của branch (thường dùng nhất)
git submodule update --remote my-project-specs

# Cách 2: kéo version mà umbrella đang trỏ (sau git pull)
git submodule update my-project-specs

# Commit lại pointer để cả team cùng version
git add my-project-specs
git commit -m "chore: update spec submodule to latest"
git push
```

> **Khác biệt:** `--remote` lấy commit mới nhất từ remote branch (khi PO vừa push); không có `--remote` → sync về đúng commit umbrella đang trỏ. `/sync` xử lý việc này tự động.

### 4.4 — Commit 2 tầng (thay đổi trong service submodule)

```bash
# Tầng 1: commit trong service submodule
cd user-service
git add specs/bdd/ src/
git commit -m "feat(FEAT-01): add BDD and implementation"
git push origin main
cd ..

# Tầng 2: update pointer ở umbrella
git add user-service
git commit -m "chore: update user-service submodule to FEAT-01"
git push
```

> ⚠️ **PHẢI push cả 2 tầng.** Push umbrella mà quên push submodule → người khác pull về bị lỗi "commit không tồn tại".
>
> **Tech-docs (API contract):** khi `setup.spec_source` được set, tech-design **luôn** nằm ở `{spec_source}/specs/tech-docs` (spec repo chung) — commit + push riêng trong spec submodule, FE/App đọc qua `/sync`. Per-service tech-docs chỉ dùng khi **không** có `spec_source`.

### 4.5 — Lỡ tay sửa file trong spec submodule (read-only)

Spec submodule là **read-only** cho dev/tester. Nếu lỡ sửa → bỏ thay đổi:

```bash
cd my-project-specs
git checkout .      # hoặc: git restore .
```

Muốn cập nhật PRD → báo PO sửa trong spec repo của họ rồi push.

### 4.6 — Troubleshooting

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| Thư mục submodule trống | Chưa init | `git submodule update --init` |
| `fatal: not a git repository` | Sai thư mục | `cd` về đúng chỗ |
| `object not found` sau pull | Push umbrella nhưng quên push submodule | Người commit submodule chạy `cd <sub> && git push` |
| Submodule ở `detached HEAD` | Bình thường khi dùng submodule | Không cần lo nếu không sửa file trong đó |
| Conflict trong submodule pointer | 2 người cùng update submodule | `git checkout --theirs <sub> && git add <sub>` |

---

## 5. Living Docs sync

Living Docs panel hiển thị trạng thái trace của toàn bộ feature. Refresh nó bằng `/validate-traces` (hoặc tự động qua `/sync`):

```bash
/validate-traces
```

- Đọc TSVs từ **tất cả** service submodule (`.trace/*.tsv` là authoritative — commit ở mỗi service).
- Cập nhật cột `dev_selftest` (pass/fail/not_run) + `dev_selftest_at`.
- Ghi canonical `trace-report.json` + bản TSV mirror → `{spec_source}/.living-docs/` (gitignored).
- Mirror thêm về `./.trace` của workspace hiện tại → panel không rỗng kể cả khi đứng trong 1 service submodule đơn lẻ.

> **Report nằm ở đâu:** canonical `trace-report.json` + TSV mirror ở **spec module** tại `{spec_source}/.living-docs/` (gitignored, regenerate bởi `/sync` hoặc `/validate-traces`). Authoritative per-service `.trace/*.tsv` vẫn commit trong mỗi service.
>
> **.gitignore:** thêm `.trace/` trong repo hiện tại **và** `.living-docs/` trong spec module — đều là mirror read-only, không commit.

### `dev_selftest` vs `qc_status` — đừng nhầm

| Cột | Set bởi | Ý nghĩa |
|---|---|---|
| `dev_selftest` (+ `dev_selftest_at`) | `/dev-run-test` | Dev tự kiểm tra code mình viết (self-check / smoke). **KHÔNG** phải coverage QC chính thức. |
| `qc_status` (+ `qc_run_at`) | `/qc-run-test` (keyed theo `@trace.verifies={UC-ID}-SC{N}`) | Kết quả QC chính thức của framework. |

Sau khi chạy `/qc-run-test` rồi `/validate-traces` (hoặc `/sync`), `qc_status` hiện lên Living Docs cạnh `dev_selftest`.

---

## 6. Workflow hằng ngày theo role

### PO — Viết tài liệu cho tính năng mới

```bash
cd my-project-spec
git pull
# Mở Claude Code tại thư mục này:
/define-product                          # khởi tạo product definition
/generate-prd {product-definition-file}
/refine-prd {prd-file}                   # review 4 lens: QA/DEV/SA/PO
/review-context {prd-file}               # chất lượng + P0 check
/review-context --fix {prd-file}         # auto-fix vấn đề nhỏ
# → update @trace.status: approved khi PRD sẵn sàng
/generate-design-spec {prd-file}         # FE/App
/generate-bdd {prd-file}                 # → web / app / system (lần lượt)

git add specs/
git commit -m "feat({domain}): add {TICKET-ID} PRD, design spec, and BDD"
git push
```

### FE/Web Dev — Nhận task mới

```bash
cd my-project-web
/sync                                    # pull + submodule + Living Docs
git add my-project-specs && git commit -m "chore: sync specs" && git push

/review-context my-project-specs/specs/prd/{domain}/{TICKET-ID}-prd.md
# → P0: @trace.domain khớp services config? @trace.status: approved?
# Đọc Web BDD do PO generate (KHÔNG tự generate BDD):
#   my-project-specs/specs/bdd/{domain}/web/{TICKET-ID}-UC*.feature

/generate-tech-docs {domain}/{TICKET-ID}-UC1
/review-tech-docs   {domain}/{TICKET-ID}-UC1

# ── BE chưa sẵn sàng ──
/generate-code {domain}/{TICKET-ID}-UC1 --phase=ui      # UI + mock adapter (tester test ngay)
/dev-gen-test  {domain}/{TICKET-ID}-UC1
/review-code   {files-changed}
/dev-run-test                                            # dev self-check (dev_selftest)
# → sau khi sign-off gate approved:
/generate-code {domain}/{TICKET-ID}-UC1 --phase=integration   # wire API thật

# ── API đã có sẵn ──
/generate-code {domain}/{TICKET-ID}-UC1                  # không cần --phase
/dev-gen-test  {domain}/{TICKET-ID}-UC1
/review-code   {files-changed}
/dev-run-test

# Commit 2 tầng (xem §4.4)
```

### BE Dev — Nhận task mới

```bash
cd my-project-be
/sync
# → cũng liệt kê "📥 New tester feedback" → /fix-bug · promote proposal · báo PO

/review-context my-project-specs/specs/prd/{domain}/{TICKET-ID}-prd.md
# Đọc System BDD: my-project-specs/specs/bdd/{domain}/system/{TICKET-ID}-UC*.feature

/generate-tech-docs {domain}/{TICKET-ID}-UC1
/review-tech-docs   {domain}/{TICKET-ID}-UC1
/generate-code      {domain}/{TICKET-ID}-UC1
/dev-gen-test       {domain}/{TICKET-ID}-UC1
/review-code        {files-changed}      # AI lặp lỗi → accept "Record as lesson?"
/dev-run-test

# Commit 2 tầng vào đúng service submodule theo domain (xem §4.4)
```

### QC — Chạy native QC pipeline (tóm tắt)

Điều kiện: BDD đã `approved`. Module `qc-playwright` (Python + pytest-playwright + Page Object).

```bash
cd my-project-web
/sync
/qc-analyze · /qc-plan · /qc-design-test · /qc-review · /qc-run-test · /qc-report
/validate-traces            # refresh dashboard → qc_status hiện trên Living Docs
```

> `/qc-run-test` ghi `qc_status` + `qc_run_at` vào trace TSV. Chi tiết từng layer: [Guide · QC Automation](../02-guides/qc-automation.md).

---

## 7. Project Lessons — không để AI lặp lỗi

Khi AI lặp đi lặp lại một lỗi, ghi lại thành **lesson** thay vì sửa thủ công mỗi lần. context-loader nạp lesson vào đầu **mọi** lệnh như ràng buộc cứng.

```bash
# Cách 1 — chủ động:
/learn "AI hay gọi repository thẳng từ controller, phải đi qua service layer"

# Cách 2 — tự động: /review-code, /fix-bug, /debug hỏi "Record as a project lesson? (Y/N)" → Y
```

- **Lưu ở:** `paths.lessons_file` (mặc định `specs/domain-knowledge/lessons-learned.md`; umbrella: `.agent/project-lessons.md` mỗi service).
- **Commit file lessons** để cả team cùng được bảo vệ — đây là **bộ nhớ dự án**, không phải fine-tune model.
- Kiểm tra `[CTX LOADED]` có dòng `Lessons: loaded — N guardrails`.

---

## 8. Câu hỏi thường gặp

**Q: Có cần cài framework vào từng service submodule không?**
A: Không. Chỉ cài ở umbrella repo. Service submodule là git repo thường chứa code + specs (+ `.agent/project-context.yaml` config).

**Q: Claude Code nên mở ở đâu?**
A: Luôn ở **umbrella repo** — có config và thấy được cả spec submodule lẫn service submodule.

**Q: Generate BDD ra file ở đâu?**
A: Phụ thuộc `@trace.domain` trong PRD. Context-loader tra `services.{domain}.specs_dir`. Vd `@trace.domain: user` → `user-service/specs/bdd/user/`.

**Q: Quên push service submodule, chỉ push umbrella, người khác báo lỗi?**
A: `cd <service> && git push origin main` — xong, người khác pull lại hết lỗi.

**Q: PO sửa PRD, có cần generate lại BDD?**
A: Chạy `/review-context {feature-file}` (B1 check) xem có coverage gap không. Có → `/review-context --resume`. Không cần viết lại toàn bộ BDD.

**Q: Mở nhiều Claude Code cho nhiều umbrella cùng lúc được không?**
A: Được. Mỗi umbrella là session riêng, hoàn toàn độc lập.

**Q: Đang ở `detached HEAD` trong submodule, có sao không?**
A: Bình thường — submodule trỏ tới một commit cụ thể. Chỉ lo nếu định sửa file (phải checkout branch trước). Chỉ đọc thì không sao.

---

*Xem thêm:* [Bug Flow](bug-flow.md) · [Publishing](publishing.md) · [02 · Guides](../02-guides/) · [03 · Concepts — Traceability](../03-concepts/traceability.md).
