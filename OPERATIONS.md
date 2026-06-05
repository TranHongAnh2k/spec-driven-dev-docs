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
  KHÔNG đụng: BDD, tech-docs, source code

FE/App dev (umbrella repo)
  ĐỌC:  my-project-specs/specs/prd/         ← PRD của PO
        my-project-specs/specs/design-spec/  ← Design Spec của PO
  VIẾT: mass-product-web/specs/bdd/          ← BDD (trong service submodule)
        mass-product-web/specs/tech-docs/    ← Tech Docs
        mass-product-web/src/                ← Source code

BE dev (umbrella repo)
  ĐỌC:  my-project-specs/specs/prd/         ← PRD của PO (không cần design-spec)
  VIẾT: user-service/specs/bdd/             ← BDD (trong service submodule)
        user-service/specs/tech-docs/        ← Tech Docs
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
    specs_dir: "mass-product-web/specs/bdd"
    tech_docs_dir: "mass-product-web/specs/tech-docs"
  payment:
    path: "mass-product-web"
    module: "nextjs"
    specs_dir: "mass-product-web/specs/bdd"
    tech_docs_dir: "mass-product-web/specs/tech-docs"
```

> **Lưu ý:** Nếu nhiều domain đều map vào cùng 1 service (như web app), tất cả đều trỏ về `mass-product-web`.

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

**`.agent/project-context.yaml` cho BE:**
```yaml
services:
  user:                              # domain "user" → user-service submodule
    path: "user-service"
    module: "java-spring"
    specs_dir: "user-service/specs/bdd"
    tech_docs_dir: "user-service/specs/tech-docs"
  order:
    path: "order-service"
    module: "java-spring"
    specs_dir: "order-service/specs/bdd"
    tech_docs_dir: "order-service/specs/tech-docs"
  payment:
    path: "payment-service"
    module: "golang"
    specs_dir: "payment-service/specs/bdd"
    tech_docs_dir: "payment-service/specs/tech-docs"
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
push to spec repo ──────────────→ git submodule update ←─── git submodule update
                                         │                         │
                                 /review-context (P0 check)  /review-context (P0 check)
                                 /generate-bdd (+ Design Spec)    /generate-bdd
                                 /generate-tech-docs         /generate-tech-docs
                                 /review-tech-docs            /review-tech-docs
                                 /generate-code               /generate-code
                                 /generate-tests              /generate-tests
                                 /review-code                 /review-code
                                 /run-tests                   /run-tests
                                 commit → push                commit → push
                                 (2-layer commit)             (2-layer commit)
```

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

#### Bước 2 — PO: Tạo Design Spec (chỉ cho FE/App)

```bash
/generate-design-spec specs/prd/auth/FEAT-01-login-prd.md
# → Hỏi platform: web hay app?
# → Output: specs/design-spec/auth/FEAT-01-design-spec-web.md

# Sau khi Designer review và confirm Figma links:
# Cập nhật @trace.status: approved trong design-spec file
```

**Push lên spec repo:**
```bash
git add specs/prd/ specs/design-spec/
git commit -m "feat(auth): add FEAT-01 PRD and design spec"
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

#### Bước 4 — Dev team: Generate BDD và code

```bash
# Mở Claude Code tại umbrella repo

# Bước 4.1 — Verify PRD trước (P0 check cho umbrella mode)
/review-context my-project-specs/specs/prd/auth/FEAT-01-login-prd.md
# → P0: @trace.domain khớp services config?
# → @trace.status: approved? (không làm tiếp nếu còn draft)

# Bước 4.2 — Generate BDD
/generate-bdd my-project-specs/specs/prd/auth/FEAT-01-login-prd.md
# → Output vào đúng service submodule theo domain routing

# Bước 4.3 — Generate Tech Docs + review
/generate-tech-docs auth/FEAT-01-UC1
/review-tech-docs   auth/FEAT-01-UC1     # kiểm tra API spec, schema

# Bước 4.4 — Generate Code + Tests
/generate-code   auth/FEAT-01-UC1
/generate-tests  auth/FEAT-01-UC1

# Bước 4.5 — Review + chạy test
/review-code     {files-changed}         # 4 lens: Security/Perf/Arch/Test
/run-tests                               # BDD pass = implement đúng spec
```

#### Bước 5 — Dev team: Commit 2 tầng

```bash
# Bước 5a: commit vào service submodule
cd mass-product-web        # hoặc user-service, v.v.
git add specs/bdd/ specs/tech-docs/ src/
git commit -m "feat(FEAT-01): add login BDD, tech-docs, implementation"
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

git add specs/
git commit -m "feat({domain}): add {TICKET-ID} PRD and design spec"
git push
```

### FE/Web Dev — Nhận task mới

```bash
cd my-project-web
git pull

# Lấy tài liệu mới nhất từ PO
git submodule update --remote my-project-specs
git add my-project-specs
git commit -m "chore: sync specs" && git push

# Mở Claude Code tại my-project-web/
/review-context my-project-specs/specs/prd/{domain}/{TICKET-ID}-prd.md
# → P0 check: @trace.domain khớp services config?
# → @trace.status: approved? — không làm tiếp nếu draft

/generate-bdd       my-project-specs/specs/prd/{domain}/{TICKET-ID}-prd.md
/generate-tech-docs {domain}/{TICKET-ID}-UC1
/review-tech-docs   {domain}/{TICKET-ID}-UC1   # verify API spec, component spec
/generate-code      {domain}/{TICKET-ID}-UC1
/generate-tests     {domain}/{TICKET-ID}-UC1
/review-code        {files-changed}            # 4 lens trước khi commit
/run-tests                                     # BDD pass = done

# Commit 2 tầng
cd mass-product-web
git add specs/ src/ && git commit -m "feat({TICKET-ID}): ..." && git push
cd ..
git add mass-product-web && git commit -m "chore: update web submodule" && git push
```

### BE Dev — Nhận task mới

```bash
cd my-project-be
git pull
git submodule update --remote my-project-specs

# Mở Claude Code tại my-project-be/
/review-context my-project-specs/specs/prd/{domain}/{TICKET-ID}-prd.md
# → @trace.status: approved? — không làm tiếp nếu draft

/generate-bdd       my-project-specs/specs/prd/{domain}/{TICKET-ID}-prd.md
/generate-tech-docs {domain}/{TICKET-ID}-UC1
/review-tech-docs   {domain}/{TICKET-ID}-UC1   # verify API spec, DB schema
/generate-code      {domain}/{TICKET-ID}-UC1
/generate-tests     {domain}/{TICKET-ID}-UC1
/review-code        {files-changed}
/run-tests

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
    specs_dir: "mass-product-web/specs/bdd"
    tech_docs_dir: "mass-product-web/specs/tech-docs"
  payment:
    path: "mass-product-web"       # ← cùng service
    module: "nextjs"
    specs_dir: "mass-product-web/specs/bdd"
    tech_docs_dir: "mass-product-web/specs/tech-docs"
  loyalty:
    path: "mass-product-web"       # ← cùng service
    module: "nextjs"
    specs_dir: "mass-product-web/specs/bdd"
    tech_docs_dir: "mass-product-web/specs/tech-docs"
```

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
*Tài liệu này được tạo bởi Spec-Driven Dev framework v0.7.0+*
