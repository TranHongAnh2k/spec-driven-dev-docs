[📚 Docs](../../README.md) › [Guides](../README.md) › [Product Owner](README.md) › Commands

# Commands Dành Cho PO/BA

| Command | Mục đích | Khi nào dùng |
|---|---|---|
| `/setup-ai-first` | Khởi tạo spec repo | 1 lần duy nhất khi bắt đầu project |
| `/define-product` | Tạo product definition | Trước khi viết PRD — capture ý tưởng tính năng |
| `/generate-prd` | Tạo PRD từ product definition | Sau khi product definition đủ rõ |
| `/refine-prd` | Review PRD qua 4 lens (QA/DEV/SA/PO) — fan-out song song + critic loop → findings đầy đủ trong 1 lần chạy | Trước khi share với dev team |
| `/review-context` | Review chất lượng + consistency của PRD (P-B check groups fan-out song song) | Sau refine, trước khi approve |
| `/review-context --fix` | Auto-fix các lỗi nhỏ trong PRD | Khi có nhiều lỗi minor tự sửa được |
| `/review-context --resume` | Apply các fix sau Review Board | Sau khi review findings |
| `/generate-design-spec` | Tạo Design Spec cho FE/App (cần Figma link node-level `?node-id=` cho từng screen) | Sau khi PRD approved, trước khi generate BDD |
| `/generate-bdd` | Tạo BDD cho web / app / system | Sau khi PRD + Design Spec approved |
| `/learn {text}` | Ghi lại lỗi AI hay lặp khi gen PRD/BDD thành guardrail | Khi AI lặp lại cùng kiểu sai trong PRD/BDD |

> Danh mục đầy đủ mọi command: [Reference › Commands](../../05-reference/commands.md).

> **Brownfield shortcut:** Nếu API đã tồn tại trên hệ thống cũ, thêm `API Source: existing` vào PRD Metadata + điền section "Existing API Contract". BDD generation sẽ dùng contract đó trực tiếp, T7 sign-off gate tự động được bỏ qua. Chi tiết: [Tình huống 11](scenarios.md#tình-huống-11--brownfield-api-đã-tồn-tại-trên-hệ-thống-cũ).

> **Project Lessons:** Nếu AI cứ lặp lại một kiểu sai khi gen PRD/BDD (vd: quên negative path, dùng sai thuật ngữ), chạy `/learn "AI hay X, đúng phải Y"` (category `prd`/`bdd`). Lesson được nạp vào đầu mọi lệnh → AI không lặp lại. Commit file lessons để cả team dùng chung.

> **Xử lý feedback từ tester:** Tester `/report-bug` và `/propose-scenario` commit vào `{spec_source}/feedback/` của spec repo. PO/Dev **thấy chúng khi chạy `/sync`** (dòng `📥 New tester feedback` liệt kê bug + proposal mới kéo về). PO review proposal:
> - **Map vào AC sẵn có** → coverage gap thật → để Dev thêm scenario vào `.feature` (hoặc regen).
> - Là **PRD change request** (behavior chưa có trong PRD) → PO thêm/sửa AC → bump version → `/generate-bdd` lại.

---

← [Product Owner Guide](README.md) · Tiếp theo: [Tình huống thực tế](scenarios.md)
