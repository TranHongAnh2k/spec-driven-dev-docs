[📚 Docs](../README.md) › Reference

# 05 · Reference

> Tra cứu nhanh: mọi slash command, schema của trace TSV, và danh sách stack module. Đây là phần "đọc khi cần" — không phải hướng dẫn tuần tự.

Dùng mục này khi bạn đã biết mình muốn làm gì và chỉ cần tra input/output, tên cột, hoặc module phù hợp với stack.

---

## Trong mục này

| Trang | Nội dung |
|-------|----------|
| [Command Reference](commands.md) | Bảng đầy đủ MỌI slash command — gom theo phase, kèm input/output/when. Cộng phần Command Internals (step architecture). |
| [Trace TSV Schema](trace-schema.md) | Header chuẩn của `.trace/{UC-ID}.tsv`, ý nghĩa từng cột (gồm `dev_selftest`, `qc_status`…), các `@trace.*` tag, và JSON report. |
| [Stack Modules](modules.md) | 15 module dưới `modules/`, cách gom theo `platform_type` (backend / web-frontend / mobile), và module QC `qc-playwright`. |

---

## Điều hướng nhanh

- **"Lệnh này nhận gì, sinh ra gì?"** → [Command Reference](commands.md)
- **"Cột `qc_status` vs `dev_selftest` khác nhau thế nào?"** → [Trace TSV Schema](trace-schema.md)
- **"Stack của tôi map sang module nào?"** → [Stack Modules](modules.md)

---

*Xem thêm:* [02 · Guides](../02-guides/) (theo vai trò) · [03 · Concepts](../03-concepts/) (kiến trúc & traceability) · [04 · Operations](../04-operations/) (vận hành).
