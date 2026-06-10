# Hướng Dẫn Vận Hành — Spec-Driven Dev

Tài liệu này dành cho toàn bộ team: PO, FE dev, BE dev, App dev.  
Mô tả cách làm việc hằng ngày với framework và git submodule.

> **Hướng dẫn chi tiết theo vai trò:**
> - PO/BA → xem [PO_GUIDE.md](PO_GUIDE.md)
> - Developer (FE/BE/App) → xem [DEV_GUIDE.md](DEV_GUIDE.md)
> - Tester / QA → xem [TESTER_GUIDE.md](TESTER_GUIDE.md)
> - Quy trình xử lý bug (PO + Dev + Tester) → xem [BUG_FLOW.md](BUG_FLOW.md)

---

## Mục Lục

1. [Kiến trúc tổng quan](#1-kiến-trúc-tổng-quan)
2. [Setup lần đầu theo role](#2-setup-lần-đầu-theo-role)
3. [Vòng đời một tính năng](#3-vòng-đời-một-tính-năng)
4. [Hướng dẫn Git Submodule](#4-hướng-dẫn-git-submodule)
5. [Workflow hằng ngày](#5-workflow-hằng-ngày)
6. [Các tình huống thực tế](#6-các-tình-huống-thực-tế)
7. [Câu hỏi thường gặp](#7-câu-hỏi-thường-gặp)

---

## 1. Kiến Trúc Tổng Quan

### Cấu trúc repo

```
my-project-spec/          ← Repo của PO (spec repo)
├── specs/
│   ├── prd/              ← PRD — platform-agnostic
│   ├── bdd/              ← BDD — sinh bởi PO, đọc bởi dev
│   │   └── {domain}/
│   │       ├── web/      ← FE/Web BDD (vocabulary: clicks, sees)
│   │       ├── app/      ← Mobile BDD (vocabulary: taps, navigates)
│   │       └── system/   ← System/BE BDD (tổng hợp từ web+app)
│   ├── design-spec/      ← Design Spec — FE/App only
│   ├── product-definition/
│   └── domain-knowledge/ ← business-dictionary, core-entities
└── .agent/               ← framework (commit vào git)

my-project-web/           ← Umbrella repo của Web team
├── .agent/project-context.yaml   ← routing config
├── my-project-specs/     ← submodule → my-project-spec (read-only)
└── mass-product-web/     ← submodule → source code web

my-project-be/            ← Umbrella repo của BE team
├── .agent/project-context.yaml
├── my-project-specs/     ← submodule → my-project-spec (read-only)
├── user-service/         ← submodule → BE microservice
└── order-service/        ← submodule → BE microservice

my-project-app/           ← Umbrella repo của App team
├── .agent/project-context.yaml
├── my-project-specs/     ← submodule → my-project-spec
└── mobile-app/           ← submodule → source code app
```

### Nguyên tắc cốt lõi

| Nguyên tắc | Mô tả |
|---|---|
| **PRD là platform-agnostic** | 1 PRD phục vụ tất cả platform. Không chứa UI details hay API specs |
| **Design Spec chỉ cho FE/App** | Chứa screen specs, component inventory, AC-UI. BE không cần |
| **Agent mở ở umbrella** | Claude Code chỉ mở ở umbrella repo — không mở trong service submodule |
| **Commit 2 tầng** | File trong submodule → commit submodule trước, update pointer ở umbrella sau |
| **Domain key phải khớp** | `@trace.domain` trong PRD phải khớp với `services` key trong umbrella config |

### Ai đọc gì — viết gì

```
PO (my-project-spec/)
  VIẾT: PRD, Design Spec, Product Definition
        BDD (web + app + system) — sinh trong spec repo
  KHÔNG đụng: tech-docs, source code

FE/App dev (umbrella repo)
  ĐỌC:  my-project-specs/specs/prd/           ← PRD của PO
        my-project-specs/specs/design-spec/    ← Design Spec của PO
        my-project-specs/specs/bdd/{domain}/web/   ← Web BDD của PO
        my-project-specs/specs/bdd/{domain}/app/   ← App BDD của PO
  VIẾT: my-project-specs/specs/tech-docs/    ← Tech Docs
        mass-product-web/src/                ← Source code

BE dev (umbrella repo)
  ĐỌC:  my-project-specs/specs/prd/           ← PRD của PO
        my-project-specs/specs/bdd/{domain}/system/  ← System BDD của PO
  VIẾT: my-project-specs/specs/tech-docs/        ← Tech Docs
        user-service/src/                    ← Source code
```

---

## 2. Setup Lần Đầu Theo Role

### 2.1 — PO / Product Team

**Chạy 1 lần khi khởi tạo project:**

```bash
cd my-project-spec

# Cài framework
npx @anhth2/spec-driven-dev-plugin --init

# Mở Claude Code tại thư mục này, chạy:
/setup-ai-first
# → chọn: 3. PO Spec repo
# → nhập domain names khi được hỏi (vd: auth, payment, loyalty)
#   ⚠️ Quan trọng: domain names này dev team sẽ dùng làm services keys
```

**Điền thông tin vào các file được tạo:**
```
.agent/project-context.yaml     ← domains list, project name
specs/domain-knowledge/
  business-dictionary.md        ← canonical terms + banned terms
  core-entities.md              ← entity glossary
```

**Commit và push:**
```bash
git add .agent/ specs/domain-knowledge/ CLAUDE.md
git commit -m "chore: init spec-driven-dev framework"
git push
```

**Thông báo cho dev teams:**
> "Spec repo ready. Domain names: `auth`, `payment`, `loyalty`.  
> Dùng các tên này làm services keys trong umbrella config của các bạn."

---

### 2.2 — Web / FE Team

**Điều kiện:** Spec repo đã được PO setup. Umbrella repo (`my-project-web`) đã có:
- `my-project-specs/` submodule → PO's spec repo
- `mass-product-web/` submodule → source code

**Chạy 1 lần:**

```bash
cd my-project-web

# Cài framework ở umbrella level (KHÔNG cài vào submodule)
npx @anhth2/spec-driven-dev-plugin --init \
  --umbrella \
  --spec-source my-project-specs \
  --services mass-product-web:nextjs

# Khởi tạo submodules nếu chưa có
git submodule update --init --recursive
```

**Review và chỉnh `.agent/project-context.yaml`:**
```yaml
services:
  auth:                              # ← phải khớp @trace.domain trong PRD
    path: "mass-product-web"         # ← tên thư mục submodule thực tế
    module: "nextjs"
    specs_dir: "mass-product-web/specs/bdd"  payment:
    path: "mass-product-web"
    module: "nextjs"
    specs_dir: "mass-product-web/specs/bdd"```

> **Lưu ý:** Nếu nhiều domain đều map vào cùng 1 service (như web app), tất cả đều trỏ về `mass-product-web`.
>
> **Tech-docs (API contract):** không khai trong `services` — context-loader tự route `tech_docs_dir → {spec_source}/specs/tech-docs` (spec repo chung). BE generate xong → commit + push lên spec repo; FE/App `/sync` để đọc.

**Mở Claude Code tại `my-project-web/` và chạy:**
```
/setup-ai-first
# → chọn: 2. Umbrella repo
```

**Commit:**
```bash
git add .agent/ .claude/
git commit -m "chore: init spec-driven-dev umbrella (web)"
git push
```

---

### 2.3 — BE Team

**Cấu trúc umbrella BE với nhiều microservices:**

```bash
cd my-project-be

# Mỗi service là 1 submodule riêng
npx @anhth2/spec-driven-dev-plugin --init \
  --umbrella \
  --spec-source my-project-specs \
  --services user:java-spring,order:java-spring,payment:golang

git submodule update --init --recursive
```

**`.agent/project-context.yaml` cho BE (umbrella level):**
```yaml
services:
  user:                              # domain "user" → user-service submodule
    path: "user-service"
    module: "java-spring"
    specs_dir: "user-service/specs/bdd"  order:
    path: "order-service"
    module: "java-spring"
    specs_dir: "order-service/specs/bdd"  payment:
    path: "payment-service"
    module: "golang"
    specs_dir: "payment-service/specs/bdd"```

> **Bắt buộc:** Mỗi service submodule cũng cần file `.agent/project-context.yaml` riêng chứa `conventions.test_command`. Framework đọc file này (Step 1.6) để chạy test đúng lệnh từ đúng thư mục khi `/run-tests` được gọi từ umbrella root.

**`.agent/project-context.yaml` cho từng service (service level):**
```yaml
# user-service/.agent/project-context.yaml
tech_stack:
  language: "Java 17"
  framework: "Spring Boot 3.2"
  module: "java-spring"

conventions:
  test_command: "mvn test"
  build_command: "mvn compile"

paths:
  trace_dir: ".trace"
```

```yaml
# payment-service/.agent/project-context.yaml
tech_stack:
  language: "Go 1.22"
  framework: "Gin"
  module: "golang"

conventions:
  test_command: "go test ./..."
  build_command: "go build ./..."

paths:
  trace_dir: ".trace"
```

---

### 2.4 — App Team

```bash
cd my-project-app

npx @anhth2/spec-driven-dev-plugin --init \
  --umbrella \
  --spec-source my-project-specs \
  --services mobile-app:flutter   # hoặc react-native, ios-swiftui, android-compose

git submodule update --init --recursive
```

### 2.5 — Tester

Tester mở Claude Code tại umbrella (có spec submodule + service submodules), không cần cài thêm framework.

```bash
git clone {tester-umbrella-url} && cd {umbrella}
/sync                          # init + pull specs/services + Living Docs
/generate-spec-manifest        # index TICKET-ID → PRD/BDD/tech-doc

# Khi test fail:
/report-bug FT-001 tài khoản khoá sau 6 lần sai, spec ghi 5
#   → ghi feedback/bug-reports/{BUG-ID}.md vào spec repo, commit+push

# Khi tìm edge case chưa có trong BDD:
/propose-scenario FT-001 login email có khoảng trắng cuối vẫn phải thành công
#   → ghi feedback/bdd-proposals/{...}.md vào spec repo, commit+push
```

Cả hai **read-only** trên spec/code chính thức — chỉ ghi vào vùng `feedback/` của spec repo dùng chung. PO/Dev thấy qua `/sync`.

---

## 3. Vòng Đời Một Tính Năng

### Tổng quan

```
PO (spec repo)                    Web/App dev (umbrella)     BE dev (umbrella)
─────────────────────────────────────────────────────────────────────────────
/define-product
/generate-prd
/refine-prd ────────────────────→ (chờ)                   (chờ)
/review-context --fix
[set @trace.status: approved]
/generate-design-spec (web/app)
[Designer sign-off]
/generate-bdd → web            ← PO gen BDD trong spec repo
/generate-bdd → app
/generate-bdd → system         ← tổng hợp từ web + app, resolve conflicts nếu có
push to spec repo ──────────────→ git submodule update ←─── git submodule update
                                         │                         │
                                 /review-context (P0 check)  /review-context (P0 check)
                                 (đọc web/app BDD)            (đọc system BDD)
                                         │                         │
                                /generate-code --phase=ui    /generate-tech-docs
                                (UI + mock adapter)           (API contract draft)
                                → Tester test FE ngay ✅          │
                                         │◄──── review tech docs ──►│
                                         │    (FE + App sign-off    │
                                         │     T7 gate — in-review) │
                                         │                         │
                                         │    [all sign-offs done] │
                                         │    tech docs: approved  │
                                         │                         │
                                                            /generate-code
                                /generate-code --phase=int   /generate-tests
                                (wire real API)              /review-code
                                 /generate-tests              /run-tests
                                 /review-code                 commit → push
                                 /run-tests                   (2-layer commit)
                                 commit → push
                                 (2-layer commit)
```

> **Giai đoạn "chờ API design"** = khoảng thời gian tech docs đang ở `in-review`.  
> BE không thể chạy `/generate-code` cho đến khi sign-off gate `approved`.  
> FE/App dùng `/generate-code --phase=ui` ngay sau khi đọc BDD — gen UI + mock adapter từ System BDD contract.  
> Tester có thể test FE ngay, không cần chờ BE.  
> Khi sign-off done → FE chạy `/generate-code --phase=integration` để wire API thật.
>
> **Nếu API đã có sẵn và đang hoạt động:** bỏ qua `--phase`, chạy `/generate-code` thẳng — gen UI + real API adapter trong 1 lần.
>
> **Nếu API tồn tại trên hệ thống cũ nhưng chưa có tài liệu:** PO khai báo `API Source: existing` trong PRD Metadata và ghi lại contract trong section "Existing API Contract". BDD gen dùng contract đó trực tiếp (bỏ qua synthesis), T7 sign-off gate tự động bị skip.

### Chi tiết từng bước

#### Bước 1 — PO: Khởi tạo tính năng

```bash
# Mở Claude Code tại my-project-spec/
/define-product
# → Agent hỏi về tính năng, output ra specs/product-definition/

/generate-prd specs/product-definition/FEAT-01-login.md
# → Output: specs/prd/auth/FEAT-01-login-prd.md
# → Đảm bảo có @trace.domain: auth trong frontmatter

/refine-prd specs/prd/auth/FEAT-01-login-prd.md
# → Review PRD qua 4 lens: QA / DEV / SA / PO
# → Phát hiện gap trước khi share với dev team

/review-context specs/prd/auth/FEAT-01-login-prd.md
# → Kiểm tra chất lượng + P0 domain check

/review-context --fix specs/prd/auth/FEAT-01-login-prd.md
# → Auto-fix các vấn đề nhỏ

# Sau khi hài lòng → đổi status sang approved:
# @trace.status: approved   (trong PRD frontmatter / metadata table)
```

#### Bước 2 — PO: Tạo Design Spec và BDD

```bash
/generate-design-spec specs/prd/auth/FEAT-01-login-prd.md
# → Hỏi platform: web hay app?
# → Output: specs/design-spec/auth/FEAT-01-design-spec-web.md

# Sau khi Designer review và confirm Figma links:
# Cập nhật @trace.status: approved trong design-spec file
```

**Generate BDD trong spec repo:**
```bash
# Bước 2a — Gen BDD cho Web
/generate-bdd specs/prd/auth/FEAT-01-login-prd.md
# → Agent hỏi: "Platform? (web / app / system)"
# → Chọn: web
# → Output: specs/bdd/auth/web/FEAT-01-UC1-login-web.feature

# Bước 2b — Gen BDD cho App (nếu project có app)
/generate-bdd specs/prd/auth/FEAT-01-login-prd.md
# → Chọn: app
# → Output: specs/bdd/auth/app/FEAT-01-UC1-login-app.feature

# Bước 2c — Gen System BDD (tổng hợp từ web + app BDDs)
/generate-bdd specs/prd/auth/FEAT-01-login-prd.md
# → Chọn: system
# → Agent đọc web/app BDDs → tổng hợp API contracts
# → Output: specs/bdd/auth/system/FEAT-01-UC1-login-system.feature
```

**Push lên spec repo:**
```bash
git add specs/prd/ specs/design-spec/ specs/bdd/
git commit -m "feat(auth): add FEAT-01 PRD, design spec, and BDD (web+app+system)"
git push
```

#### Bước 3 — Dev team: Lấy tài liệu mới

```bash
# Từ umbrella repo (my-project-web/ hoặc my-project-be/)
git pull
git submodule update --remote my-project-specs
git add my-project-specs
git commit -m "chore: update spec submodule (FEAT-01)"
git push
```

#### Bước 4 — Dev team: Đọc BDD và implement

```bash
# Mở Claude Code tại umbrella repo

# Bước 4.1 — Verify PRD + lấy BDD từ spec submodule
/review-context my-project-specs/specs/prd/auth/FEAT-01-login-prd.md
# → P0: @trace.domain khớp services config?
# → @trace.status: approved? (không làm tiếp nếu còn draft)

# Bước 4.2 — Đọc BDD đã được PO generate (KHÔNG tự generate BDD)
# FE/Web team đọc:
#   my-project-specs/specs/bdd/auth/web/FEAT-01-UC1-login-web.feature
# App team đọc:
#   my-project-specs/specs/bdd/auth/app/FEAT-01-UC1-login-app.feature
# BE team đọc:
#   my-project-specs/specs/bdd/auth/system/FEAT-01-UC1-login-system.feature

# Bước 4.3 — Generate Tech Docs + review
/generate-tech-docs auth/FEAT-01-UC1
/review-tech-docs   auth/FEAT-01-UC1     # kiểm tra API spec, schema

# Bước 4.4 — Generate Code + Tests
# FE/App (2 phases — không cần chờ BE):
/generate-code --phase=ui    auth/FEAT-01-UC1  # gen UI + mock adapter, tester test ngay
# [chờ T7 sign-off gate hoàn tất]
/generate-code --phase=integration auth/FEAT-01-UC1  # wire API thật sau sign-off

# BE (1 phase — sau khi sign-off approved):
/generate-code   auth/FEAT-01-UC1
/generate-tests  auth/FEAT-01-UC1

# Bước 4.5 — Review + chạy test
/review-code     {files-changed}         # 4 lens: Security/Perf/Arch/Test
/run-tests                               # tests pass = implement đúng spec

# Bước 4.6 — Sync Living Docs panel (umbrella mode)
/validate-traces
# → Reads TSVs từ tất cả service submodule (.trace/ của từng service)
# → Copies về umbrella .trace/{service}/{UC-ID}.tsv
# → Writes merged trace-report.json → umbrella .trace/
# → Living Docs panel cập nhật ngay
```

> **Umbrella .gitignore:** Thêm `.trace/` vào `.gitignore` tại umbrella root — đây là mirror read-only, không commit.
> Authoritative trace state ở từng service submodule và được commit ở đó.

#### Bước 5 — Dev team: Commit 2 tầng

```bash
# Bước 5a: commit vào service submodule
cd mass-product-web        # hoặc user-service, v.v.
git add specs/bdd/ src/
git commit -m "feat(FEAT-01): add login BDD, implementation"
# tech-docs (API contract) → commit + push riêng trong spec submodule
git push origin main

# Bước 5b: update pointer ở umbrella
cd ..                      # về umbrella root
git add mass-product-web   # hoặc user-service
git commit -m "chore: update mass-product-web to FEAT-01"
git push
```

---

## 4. Hướng Dẫn Git Submodule

### 4.1 — Clone lần đầu

```bash
# Clone kèm tất cả submodules luôn
git clone --recurse-submodules https://gitlab.company.com/my-project-web.git

# Hoặc nếu đã clone rồi mà submodule trống:
cd my-project-web
git submodule update --init --recursive
```

### 4.2 — Kiểm tra trạng thái submodule

```bash
git submodule status
```

| Ký hiệu | Ý nghĩa | Cách xử lý |
|---|---|---|
| `abc1234 my-project-specs` | Bình thường | — |
| `+abc1234 my-project-specs` | Umbrella có thay đổi chưa commit | `git add my-project-specs && git commit` |
| `-abc1234 my-project-specs` | Chưa init | `git submodule update --init` |
| `Uabc1234 my-project-specs` | Conflict | Giải quyết conflict rồi commit |

### 4.3 — Cập nhật spec submodule (lấy PRD mới từ PO)

```bash
# Cách 1: Kéo về version mới nhất của branch (thường dùng nhất)
git submodule update --remote my-project-specs

# Cách 2: Kéo về version mà umbrella đang trỏ (sau khi git pull)
git submodule update my-project-specs

# Commit lại pointer để cả team cùng version
git add my-project-specs
git commit -m "chore: update spec submodule to latest"
git push
```

> **Khác biệt quan trọng:**
> - `--remote` → lấy commit mới nhất từ remote branch (khi PO vừa push)
> - Không có `--remote` → sync về đúng commit mà umbrella đang trỏ (sau khi `git pull`)

> **`--remote` lấy branch nào?** Theo thứ tự: `submodule.<spec>.branch` trong `.gitmodules` → nếu không khai báo thì lấy **default branch của remote** (`origin/HEAD`, thường `main`/`master`). Nếu PO làm việc trên branch khác (vd `develop`), phải khai báo một lần cho cả team:
> ```bash
> git config -f .gitmodules submodule.my-project-specs.branch develop
> git add .gitmodules && git commit -m "chore: pin spec submodule branch = develop"
> ```
> **Lệnh `/sync` xử lý việc này tự động** — đọc `.gitmodules`, hoặc nhận override một lần qua `/sync develop`. Service submodules không bị ảnh hưởng (luôn checkout đúng SHA umbrella ghi, không phụ thuộc branch).

### 4.4 — Commit thay đổi trong service submodule

**Quy trình chuẩn (2 tầng):**

```bash
# Tầng 1: commit trong service submodule
cd user-service
git status                    # kiểm tra files thay đổi
git add specs/bdd/ src/
git commit -m "feat(FEAT-01): add BDD and implementation"
git push origin main
cd ..

# Tầng 2: update pointer ở umbrella
git add user-service          # update submodule pointer
git status                    # kiểm tra: "modified: user-service (new commits)"
git commit -m "chore: update user-service submodule to FEAT-01"
git push
```

> ⚠️ **Lưu ý:** PHẢI push cả 2 tầng. Nếu chỉ push umbrella mà chưa push submodule
> → người khác pull về sẽ bị lỗi "commit không tồn tại".

### 4.5 — Lỡ tay sửa file trong spec submodule (read-only)

```bash
# Submodule spec là read-only cho dev team — không được commit vào đây
# Nếu lỡ sửa → bỏ thay đổi:

cd my-project-specs
git checkout .                # bỏ tất cả thay đổi chưa staged
# hoặc
git restore .
```

Nếu muốn cập nhật PRD → báo PO, để PO sửa trong spec repo của họ rồi push.

### 4.6 — Troubleshooting thường gặp

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| Thư mục submodule trống | Chưa init | `git submodule update --init` |
| `fatal: not a git repository` | Sai thư mục | `cd` về đúng chỗ |
| `object not found` sau pull | Push umbrella nhưng quên push submodule | Người đã commit submodule chạy `cd submodule && git push` |
| Submodule ở `detached HEAD` | Bình thường khi dùng submodule | Không cần lo nếu không sửa file trong đó |
| Conflict trong submodule pointer | 2 người cùng update submodule | `git checkout --theirs submodule-name && git add submodule-name` |

### 4.7 — Hai loại "update" — đừng nhầm

Có 2 thứ cần update, **nguồn khác nhau, tần suất khác nhau**:

| | `/sync` | `/update-framework` |
|---|---|---|
| **Update cái gì** | Nội dung dự án: code/specs trong submodule + Living Docs | Bản thân framework: `.agent/commands/`, steps/, modules/ |
| **Nguồn** | Git remote của các submodule | npm registry |
| **Tần suất** | Mỗi sáng / trước khi work | Khi có version framework mới (hiếm) |
| **Có đụng project-context.yaml / CLAUDE.md không?** | Không | Không (chỉ refresh file framework) |

```bash
# Update nội dung dự án — chạy thường xuyên
/sync

# Nâng cấp framework — chạy khi có bản mới
/update-framework
# → đọc .agent/FRAMEWORK_VERSION, so npm, chạy npx @latest --init
# → review git diff .agent/ rồi commit
```

> **Umbrella mode:** `/update-framework` chỉ cần chạy ở umbrella root. Service submodule chỉ chứa `.agent/project-context.yaml` (config), không có command files → không cần nâng cấp riêng.

### 4.8 — Project Lessons — không để AI lặp lỗi

Khi AI lặp đi lặp lại một lỗi trong dự án, ghi lại thành **lesson** thay vì sửa thủ công mỗi lần. context-loader nạp lesson vào đầu **mọi** lệnh như ràng buộc cứng → AI không tái phạm.

```bash
# Cách 1 — chủ động
/learn "AI hay gọi repository thẳng từ controller, phải đi qua service layer"

# Cách 2 — tự động: /review-code, /fix-bug, /debug hỏi "Record as a project lesson? (Y/N)"
#          khi phát hiện lỗi AI hay lặp → chọn Y
```

- **Lưu ở:** `paths.lessons_file` (mặc định `specs/domain-knowledge/lessons-learned.md`; umbrella: `.agent/project-lessons.md` mỗi service)
- **Commit file lessons** để cả team cùng được bảo vệ — đây là **bộ nhớ dự án**, không phải fine-tune model
- Kiểm tra `[CTX LOADED]` có dòng `Lessons: loaded — N guardrails`

---

## 5. Workflow Hằng Ngày

### PO — Viết tài liệu cho tính năng mới

```bash
cd my-project-spec
git pull

# Mở Claude Code tại thư mục này
/define-product                          # khởi tạo product definition
/generate-prd {product-definition-file}  # generate PRD
/refine-prd {prd-file}                   # review 4 lens: QA/DEV/SA/PO
/review-context {prd-file}               # kiểm tra chất lượng + P0 check
/review-context --fix {prd-file}         # auto-fix vấn đề nhỏ
# → update @trace.status: approved khi PRD sẵn sàng
/generate-design-spec {prd-file}         # generate design spec (FE/App)
/generate-bdd {prd-file}                 # → chọn: web  (FE BDD)
/generate-bdd {prd-file}                 # → chọn: app  (App BDD, nếu có)
/generate-bdd {prd-file}                 # → chọn: system (tổng hợp từ web+app)

git add specs/
git commit -m "feat({domain}): add {TICKET-ID} PRD, design spec, and BDD"
git push
```

### FE/Web Dev — Nhận task mới

```bash
cd my-project-web

# Sync toàn bộ (git pull + submodule + Living Docs) — 1 lệnh duy nhất
/sync

# Commit pointer submodule nếu có thay đổi mới từ PO
git add my-project-specs && git commit -m "chore: sync specs" && git push

# Mở Claude Code tại my-project-web/
/review-context my-project-specs/specs/prd/{domain}/{TICKET-ID}-prd.md
# → P0 check: @trace.domain khớp services config?
# → @trace.status: approved? — không làm tiếp nếu draft

# Đọc Web BDD do PO đã generate (KHÔNG tự generate BDD)
# BDD nằm tại: my-project-specs/specs/bdd/{domain}/web/{TICKET-ID}-UC*.feature

/generate-tech-docs {domain}/{TICKET-ID}-UC1
/review-tech-docs   {domain}/{TICKET-ID}-UC1   # verify API spec, component spec

# ── Trường hợp BE chưa sẵn sàng ──────────────────────────────────────────────
/generate-code {domain}/{TICKET-ID}-UC1 --phase=ui   # UI + mock adapter
# → tester test được ngay mà không cần chờ BE
/generate-tests {domain}/{TICKET-ID}-UC1
/review-code    {files-changed}
/run-tests
# → push, tester test với mock data
# → sau khi sign-off gate approved:
/generate-code {domain}/{TICKET-ID}-UC1 --phase=integration  # wire API thật
/run-tests

# ── Trường hợp API đã có sẵn / sẵn sàng ────────────────────────────────────
/generate-code  {domain}/{TICKET-ID}-UC1   # không cần --phase
/generate-tests {domain}/{TICKET-ID}-UC1
/review-code    {files-changed}
/run-tests
# ─────────────────────────────────────────────────────────────────────────────

# Commit 2 tầng
cd mass-product-web
git add specs/ src/ && git commit -m "feat({TICKET-ID}): ..." && git push
cd ..
git add mass-product-web && git commit -m "chore: update web submodule" && git push
```

### BE Dev — Nhận task mới

```bash
cd my-project-be

# Sync toàn bộ (git pull + submodule + Living Docs) — 1 lệnh duy nhất
/sync
# → cũng liệt kê "📥 New tester feedback" (bug report / scenario proposal mới từ tester)
#   → /fix-bug {BUG-ID} · promote proposal vào BDD · hoặc báo PO sửa PRD

# Mở Claude Code tại my-project-be/
/review-context my-project-specs/specs/prd/{domain}/{TICKET-ID}-prd.md
# → @trace.status: approved? — không làm tiếp nếu draft

# Đọc System BDD do PO đã generate (KHÔNG tự generate BDD)
# BDD nằm tại: my-project-specs/specs/bdd/{domain}/system/{TICKET-ID}-UC*.feature

/generate-tech-docs {domain}/{TICKET-ID}-UC1
/review-tech-docs   {domain}/{TICKET-ID}-UC1   # verify API spec, DB schema
/generate-code      {domain}/{TICKET-ID}-UC1
/generate-tests     {domain}/{TICKET-ID}-UC1
/review-code        {files-changed}            # nếu thấy lỗi AI hay lặp → accept "Record as lesson?"
/run-tests
# /learn "AI hay X, đúng phải Y"               # ghi guardrail khi AI lặp lỗi (tùy chọn)

# Commit 2 tầng (vào đúng service submodule theo domain)
cd user-service    # hoặc order-service tùy domain
git add specs/ src/ && git commit -m "feat({TICKET-ID}): ..." && git push
cd ..
git add user-service && git commit -m "chore: update user-service submodule" && git push
```

---

## 6. Các Tình Huống Thực Tế

### Tình huống 1 — PO chạy trước, dev repo chưa có

PO viết PRD bình thường trong spec repo. Không cần đợi dev team.

```
PO viết: specs/prd/auth/FEAT-01-login-prd.md (@trace.domain: auth)
         specs/prd/auth/FEAT-02-register-prd.md (@trace.domain: auth)
         specs/design-spec/auth/FEAT-01-design-spec-web.md
```

Khi dev team join:
1. Setup umbrella → add spec repo làm submodule
2. Dùng `auth` làm services key trong config
3. `git submodule update --init --remote` → pull hết PRDs đã có
4. Chạy `/review-context` từng PRD để verify P0 → generate BDD

### Tình huống 2 — Thêm platform mới (vd: BE join sau khi Web đã chạy)

PRD **không cần sửa** vì đã platform-agnostic.

```bash
# BE team tự setup umbrella
cd my-project-be
npx @anhth2/spec-driven-dev-plugin --init \
  --umbrella \
  --spec-source my-project-specs \
  --services user:java-spring

# Pull toàn bộ PRDs đã có
git submodule update --init --remote my-project-specs

# Generate BDD từ PRDs hiện có
/generate-bdd my-project-specs/specs/prd/auth/FEAT-01-login-prd.md
```

Web team không bị ảnh hưởng gì.

### Tình huống 3 — PRD cần update sau khi đã generate BDD

```bash
# PO: cập nhật PRD
# Đổi status về draft, bump version
@trace.status: draft
@trace.version: 1.0 → 1.1

# Sửa nội dung → review → approved → push

# Dev team: pull spec mới
git submodule update --remote my-project-specs

# Re-review PRD
/review-context my-project-specs/specs/prd/auth/FEAT-01-login-prd.md

# Nếu BDD cần update:
/review-context user-service/specs/bdd/auth/FEAT-01-UC1.feature
# → Xem có coverage gap không (B1 check)
# → /review-context --resume nếu cần fix
```

### Tình huống 4 — `@trace.domain` không khớp services config

```
⚠️  ROUTING WARNING (từ /review-context P0 check):
  F001 [critical] P0.2 — @trace.domain: "users" không khớp services keys: [user, order, payment]
```

Cách xử lý:
- **Option A:** PO sửa PRD: `@trace.domain: users` → `@trace.domain: user` → push
- **Option B:** Dev team thêm entry vào services config: `users:` mapping → bao gồm cả `user:` để giữ backward compat

### Tình huống 5 — Chỉ có 1 FE service nhưng nhiều domain

Web app có nhiều domain (auth, payment, loyalty) nhưng tất cả đều trong cùng 1 service:

```yaml
# my-project-web/.agent/project-context.yaml
services:
  auth:
    path: "mass-product-web"       # ← cùng service
    module: "nextjs"
    specs_dir: "mass-product-web/specs/bdd"  payment:
    path: "mass-product-web"       # ← cùng service
    module: "nextjs"
    specs_dir: "mass-product-web/specs/bdd"  loyalty:
    path: "mass-product-web"       # ← cùng service
    module: "nextjs"
    specs_dir: "mass-product-web/specs/bdd"```

Tất cả domain đều route vào cùng 1 service — BDD phân biệt nhau bằng thư mục con (`specs/bdd/auth/`, `specs/bdd/payment/`).

---

## 7. Câu Hỏi Thường Gặp

**Q: Tôi có cần cài framework vào từng service submodule không?**  
A: Không. Framework chỉ cần cài ở umbrella repo. Service submodule chỉ là git repo thông thường chứa code và specs.

**Q: Claude Code nên mở ở đâu?**  
A: Luôn mở ở **umbrella repo** (không mở trong service submodule). Umbrella có config (`.agent/project-context.yaml`) và có thể thấy cả spec submodule lẫn service submodule.

**Q: Khi generate-bdd, BDD file ra ở đâu?**  
A: Phụ thuộc vào `@trace.domain` trong PRD. Context-loader tra `services.{domain}.specs_dir` trong config. Ví dụ `@trace.domain: user` → BDD ra `user-service/specs/bdd/user/`.

**Q: Tôi quên push service submodule, chỉ push umbrella. Giờ người khác báo lỗi. Phải làm gì?**  
A:
```bash
cd user-service
git push origin main
# Xong — người khác pull lại sẽ hết lỗi
```

**Q: PO sửa PRD, tôi có cần generate lại BDD không?**  
A: Chạy `/review-context {feature-file}` (B1 check) để kiểm tra PRD mới có tạo coverage gap không. Nếu có → `/review-context --resume` để generate thêm scenarios. Không cần viết lại toàn bộ BDD.

**Q: Tôi có thể mở nhiều Claude Code cho nhiều umbrella cùng lúc không?**  
A: Có. Mỗi umbrella là một session riêng. Web dev mở `my-project-web/`, BE dev mở `my-project-be/` — hoàn toàn độc lập.

**Q: Spec submodule cần update thường xuyên không?**  
A: Chạy `git submodule update --remote my-project-specs` mỗi khi PO thông báo có PRD mới. Không cần update liên tục — chỉ khi bắt đầu task mới.

**Q: Tôi đang ở detached HEAD trong submodule, có sao không?**  
A: Bình thường. Submodule được trỏ đến một commit cụ thể, không phải branch. Chỉ lo nếu bạn định sửa file trong đó (phải checkout branch trước). Nếu chỉ đọc → hoàn toàn bình thường.

---

---

*Xem thêm:* [PO_GUIDE.md](PO_GUIDE.md) — tình huống chi tiết dành riêng cho PO/BA  
*Xem thêm:* [SETUP_GUIDE.md](SETUP_GUIDE.md) — hướng dẫn cài đặt framework  
*Tài liệu này được tạo bởi Spec-Driven Dev framework v0.8.0+*
