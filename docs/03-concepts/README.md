[📚 Docs](../README.md) › Concepts

# Concepts

Hiểu **cách framework được xây dựng và vận hành** — kiến trúc nhiều lớp, pipeline các phase, và hệ thống traceability. Đọc section này khi bạn muốn biết *vì sao* mọi thứ hoạt động như vậy, không chỉ *cách dùng* lệnh.

## Mục lục (this section)

- [architecture.md](architecture.md) — 6 lớp (Protection → Output), module plug-in system, build pipeline (`*.tmpl` → `*.md` → `core/`), directory map, sub-agent orchestration, hook data-protection.
- [pipeline.md](pipeline.md) — các phase Discovery → PRD → Design-Spec → BDD → Tech-Docs → Code → Dev self-check → QC automation → Tester feedback; review gates; và step-architecture model (gate / context-loader / spawn-agent / report-footer) đằng sau mỗi command.
- [traceability.md](traceability.md) — `@trace.*` tags, trace TSV, hai tín hiệu `dev_selftest` vs `qc_status`, Living Docs (canonical trong spec-module + panel mirror), và `/validate-traces`.

## Đọc gì trước?

1. Muốn cái nhìn tổng thể? → [architecture.md](architecture.md) — file "đọc trước khi đọc bất kỳ file chi tiết nào".
2. Muốn hiểu luồng làm việc end-to-end? → [pipeline.md](pipeline.md).
3. Muốn hiểu coverage / drift / Living Docs? → [traceability.md](traceability.md).

> Tra cứu chi tiết schema và module thì sang [05 · Reference](../05-reference/) — [trace-schema.md](../05-reference/trace-schema.md) và [modules.md](../05-reference/modules.md).
