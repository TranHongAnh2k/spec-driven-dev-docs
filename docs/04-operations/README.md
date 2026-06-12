[📚 Docs](../README.md) › Operations

# 04 · Operations

> Vận hành hằng ngày với framework: đồng bộ nội dung & nâng cấp framework, quy trình xử lý bug, và publish package lên npm.

Mục này dành cho cả team (PO · Dev · Tester · QC) và người maintain framework.

---

## Trong mục này

| Trang | Nội dung |
|-------|----------|
| [Sync & Update](sync-and-update.md) | `/sync` vs `/update-framework`, umbrella mode, đồng bộ Living Docs, workflow hằng ngày theo role |
| [Bug Flow](bug-flow.md) | Phân loại bug theo spec layer (Code/BDD/PRD/Design/Env), 6 case, format thông báo, checklist đóng bug |
| [Publishing](publishing.md) | Build + publish `@anhth2/spec-driven-dev-plugin` lên npm, versioning, transfer ownership |

---

## Hai loại "update" — đừng nhầm

| | `/sync` | `/update-framework` |
|---|---|---|
| Update cái gì | Nội dung dự án: code/specs trong submodule + Living Docs | Bản thân framework: commands, steps, modules |
| Nguồn | Git remote của submodule | npm registry |
| Tần suất | Mỗi sáng / trước khi work | Khi có version framework mới (hiếm) |

Chi tiết: [Sync & Update](sync-and-update.md).

---

*Xem thêm:* [02 · Guides](../02-guides/) (theo vai trò) · [03 · Concepts](../03-concepts/) (kiến trúc) · [05 · Reference](../05-reference/) (tra cứu lệnh).
