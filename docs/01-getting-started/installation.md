[📚 Docs](../README.md) › [Getting Started](README.md) › Installation

# Installation

Cài đặt framework vào project và (tùy chọn) VS Code extension.

## Mục lục

- [Prerequisites](#prerequisites)
- [Cài framework](#cài-framework)
- [Kiểm tra cài đặt](#kiểm-tra-cài-đặt)
- [Upgrade](#upgrade)
- [QC automation stack (tùy chọn)](#qc-automation-stack-tùy-chọn)
- [VS Code extension (khuyến nghị)](#vs-code-extension-khuyến-nghị)
- [Uninstall](#uninstall)

## Prerequisites

| Tool | Version | Link |
|------|---------|------|
| Node.js | bất kỳ (check: `node -v`) | [nodejs.org](https://nodejs.org) |
| Claude Code CLI | Latest | [claude.ai/code](https://claude.ai/code) |
| VS Code | ≥ 1.85 | [code.visualstudio.com](https://code.visualstudio.com) |
| Git | bất kỳ | |

> Claude Code cần subscription (Claude Pro / Team / API key).

## Cài framework

Chạy từ **thư mục root của project**. Cách khuyến nghị là `--init` — cài framework vào `.agent/` (commit vào git, cả team dùng chung) và tạo shortcut trong `.claude/commands/`.

```bash
# Single-service project:
npx @anhth2/spec-driven-dev-plugin --init --module java-spring

# Multi-service monorepo (cài vào từng subfolder trong 1 lệnh):
npx @anhth2/spec-driven-dev-plugin --init \
  --services backend:java-spring,web-admin:react,app-mobile:flutter
```

Kết quả:
- `.agent/` — toàn bộ framework files (commit vào git, shared với team)
- `.claude/commands/` — shortcut trỏ về `.agent/commands/`
- `.agent/FRAMEWORK_VERSION` — tracking version để upgrade

```bash
git add .agent/ .claude/commands/
git commit -m "chore: init spec-driven-dev"
```

> **Multi-repo / umbrella?** Xem hướng dẫn umbrella setup chi tiết trong [Operations › Sync & Update §4 Umbrella mode](../04-operations/sync-and-update.md#4-umbrella-mode--git-submodule).

### Legacy install (global / per-project)

```bash
npx @anhth2/spec-driven-dev-plugin           # global: ~/.claude/commands/
npx @anhth2/spec-driven-dev-plugin --project # project: ./.claude/commands/
```

## Kiểm tra cài đặt

Mở Claude Code tại project, gõ `/` — bạn sẽ thấy các lệnh:

```
/setup-ai-first
/define-product
/generate-prd
/generate-bdd
...
```

> Không thấy lệnh? Chạy lại lệnh cài (global lưu tại `~/.claude/commands/`, trên Windows: `ls "$env:USERPROFILE\.claude\commands\"`).

## Upgrade

Từ **trong Claude Code** (khuyến nghị — check version, xử lý umbrella mode, review diff):

```
/update-framework
```

Hoặc từ terminal:

```bash
bash scripts/upgrade.sh
# hoặc:
npx @anhth2/spec-driven-dev-plugin@latest --init
git diff .agent/ && git add .agent/ && git commit -m "chore: upgrade framework"
```

> Chỉ upgrade framework command files. `project-context.yaml`, `CLAUDE.md`, domain-knowledge, và `.trace/` không bao giờ bị ghi đè. Để sync *content* của project (submodule code/specs) dùng `/sync`.

## QC automation stack (tùy chọn)

Bộ QC suite chính thức (`/qc-*`) là **native** trong framework. Hai lệnh cuối — `/qc-run-test` và `/qc-report` — dùng stack module **`qc-playwright`**, **độc lập với dev module**. Cài riêng:

```bash
# 1. Python 3 + pytest-playwright
pip install pytest-playwright

# 2. Cài browsers
python3 -m playwright install
```

`qc-playwright` = Python + pytest-playwright + Page Object (output: Playwright Trace + pytest-html). Chi tiết pipeline: [../02-guides/qc-automation.md](../02-guides/qc-automation.md).

## VS Code extension (khuyến nghị)

**Spec Driven Dev** là VS Code extension với 2 panels. Không bắt buộc nhưng khuyến nghị. VS Code tự cập nhật khi có version mới.

```bash
code --install-extension edupia-team.spec-driven-dev-team
```

Hoặc: `Ctrl+Shift+P` → **"Extensions: Install from Marketplace"** → search **Spec Driven Dev**.

### Panel 1 — Review Board

Đọc `*-findings.yaml` từ `.agent/review/` — hỗ trợ findings từ mọi review command (`/refine-prd`, `/review-context`, `/review-tech-docs`).

- Lens tabs: All / QA / DEV / SA / PO hoặc PRD / BDD / TECH.
- Mỗi finding có badge `⚡ auto-fix` / `👤 human`.
- 4 actions: Accept · Modify (có note) · Defer · Reject — kèm progress bar, full-text search.
- **Smart Apply** spawn terminal chạy đúng lệnh `--resume`.

Mở Review Board: sidebar panel (Activity Bar), hoặc right-click file `*-findings.yaml` → "Open Review Board", hoặc Command Palette. Nếu panel trống: file phải có đuôi `-findings.yaml`, nằm trong `.agent/review/`, và đóng hẳn rồi mở lại VS Code.

### Panel 2 — Living Documentation

Đọc `.trace/*.tsv` — dashboard traceability health toàn project.

- Stat cards: PRDs, Use Cases, Scenarios, Code Cov%, Test Cov%, Drift, Gap.
- Drill-down: PRD → UC → per-scenario table (Spec ver, Gen ver, Code, Tests, `dev_selftest`, `qc_status`, Status).
- Status badges: ✅ OK · ⚠️ DRIFT · 🔴 GAP · — UNTRACKED. Filter + search + live reload.

Mở: `Ctrl+Shift+P` → **"Spec Driven Dev: Open Living Documentation"**. Mở được cả ở umbrella root lẫn trong một service submodule riêng lẻ. Nếu trống, chạy `/generate-bdd` cho ≥1 feature để tạo file `.trace/{UC-ID}.tsv` đầu tiên.

> Report canonical sinh vào spec module tại `{spec_source}/.living-docs/` (gitignored); bản mirror cục bộ ở `./.trace`. Chi tiết traceability: [core-concepts.md](core-concepts.md) và [../03-concepts/traceability.md](../03-concepts/traceability.md).

## Uninstall

**Mac/Linux:**
```bash
rm -rf .agent/ .claude/commands/
```

**Windows (PowerShell):**
```powershell
Remove-Item -Recurse -Force .agent, .claude\commands
```

---

Cài xong? → [quickstart.md](quickstart.md) để chạy feature đầu tiên.
