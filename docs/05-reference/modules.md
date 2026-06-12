[📚 Docs](../README.md) › [Reference](README.md) › Stack Modules

# Stack Modules

> Mỗi module dưới `modules/` ship một `stack-profile.yaml` chứa layer pattern, naming rule, và test pattern riêng cho framework đó. Framework detect platform type từ `tech_stack.module` (hoặc `active_module` trong multi-service) và adapt mọi artifact sinh ra.

---

## Mục lục

- [Danh sách module](#danh-sách-module)
- [Platform type grouping](#platform-type-grouping)
- [qc-playwright — QC automation stack](#qc-playwright--qc-automation-stack)

---

## Danh sách module

15 module (xác minh bằng listing thư mục `modules/`):

| Module | Language / Framework | Platform type |
|--------|----------------------|---------------|
| `java-spring` | Java + Spring Boot | backend |
| `golang` | Go + Gin/Echo | backend |
| `dotnet` | C# + .NET / ASP.NET Core | backend |
| `php-laravel` | PHP + Laravel | backend |
| `context-engineering` | Generic AI/LLM projects | backend |
| `react` | TypeScript + React | web-frontend |
| `nextjs` | TypeScript + Next.js | web-frontend |
| `vue` | TypeScript + Vue 3 | web-frontend |
| `nuxt` | TypeScript + Nuxt 3 + @nuxt/ui | web-frontend |
| `angular` | TypeScript + Angular | web-frontend |
| `flutter` | Dart + Flutter | mobile |
| `react-native` | TypeScript + React Native / Expo | mobile |
| `ios-swiftui` | Swift + SwiftUI | mobile |
| `android-compose` | Kotlin + Jetpack Compose | mobile |
| `qc-playwright` | Python + pytest-playwright + Page Object | QC / E2E (xem dưới) |

---

## Platform type grouping

`platform_type` được suy ra từ module và quyết định cách adapt BDD vocabulary, tech-design sections, và test templates.

| `platform_type` | Modules |
|-----------------|---------|
| `backend` | `java-spring`, `golang`, `dotnet`, `php-laravel`, `context-engineering` |
| `web-frontend` | `react`, `nextjs`, `vue`, `nuxt`, `angular` |
| `mobile` | `flutter`, `react-native`, `ios-swiftui`, `android-compose` |

Ví dụ adapt theo platform type:

- **BDD vocabulary** — backend dùng "submits a request" / "receives response"; web dùng "clicks" / "types into" / "navigates to"; mobile dùng "taps" / "enters" / "opens".
- **Tech-design sections** — backend: §4 API Endpoints / §5 Service Flow / §6 Data Model…; web-frontend: §4 Screen/Component Breakdown / §5 State Management…; mobile: §4 Screen/Widget Breakdown / §5 State & Data Flow….
- **Test templates** — Java → JUnit 5 + Mockito + `@WebMvcTest`; web → Vitest/Jest + Testing Library + Playwright/Cypress; Flutter → `testWidgets`; iOS → `XCTestCase`; Android → Compose rule + `@HiltAndroidTest`.

---

## qc-playwright — QC automation stack

`qc-playwright` là **stack QC automation độc lập** với dev implementation module. Nó power native QC pipeline (`/qc-run-test`, `/qc-report`) và được chọn qua `tech_stack.qc_module` hoặc per `/qc-*` run — không phụ thuộc vào stack dev (java-spring, react, flutter…). Per-layer guides nằm ở `skills/qc/<stage>/`. Xem [../02-guides/qc-automation.md](../02-guides/qc-automation.md).

**Stack** (từ `modules/qc-playwright/stack-profile.yaml`): Python 3 + pytest-playwright + Page Object Model. Report = Playwright Trace + pytest-html (**không** dùng Allure, không hand-written dashboard, không `record_video`).

**Build commands:**

| Mục đích | Command |
|----------|---------|
| Test | `python3 -m pytest` |
| E2E | `python3 -m pytest -m e2e` |
| Report | `python3 -m pytest --html=reports/<feature>/report.html --self-contained-html` |
| Show trace | `python3 -m playwright show-trace <test-results/<nodeid>/trace.zip>` |

**Key architecture rules:**
- **Markdown-first** — không sinh Python cho tới khi có `.Test.md` đã review cho feature.
- Không hard-code URL/credential/timeout — đọc từ `Env.*` và `CONFIG[...]`.
- Không `time.sleep()` — dùng Playwright auto-wait / `expect()`.
- Mỗi test độc lập qua pytest-playwright fixtures (`page` / `logged_in_page` / …).
- Page Object extends slim `BasePage`; tách 3 layer: locators `_x()`, actions `verb_noun()`, assertions `assert_x()` dùng `expect()`.
- Locator priority: `data-testid` → role → label/text → CSS → tránh XPath.
- Group test theo `(role, account)` để login/logout không interleave giữa các role.
- Cover 100% TC trong `.Test.md` — mỗi TC kết thúc Pass/Fail/Skip, không để Draft.

**Naming:** test case id `TC_<FEATURE>_<NNN>` · test class `TestFeatureHappyCase` · test function `test_TC<NNN>_<snake_case>` · page object `<feature>_page.py` (`<Feature>Page` extends `BasePage`).

**Folder structure:**
```
docs/<project>/<feature>/   ← test-case Markdown (.Test.md) — source of truth
pages/                      ← Page Object Model (base_page.py + <feature>_page.py)
tests/                      ← pytest scripts, 1-1 với docs/ (conftest.py + test_<feature>.py)
utils/                      ← config_loader, logger, steps, test_ordering, report helpers
test_data/                  ← JSON datasets
config/config.yaml          ← browser, timeout, video/screenshot/trace toggles
reports/  test-results/     ← generated (gitignored): html report, trace.zip, screenshots
```

**Test layers:** functional (gui-screen / gui-feature / api) · integration (api/db/gui/kafka) · e2e (journey) · non-functional · exploratory.

**Trace tags** (map QC test về scenario, drive `qc_status`):
```python
# @trace.verifies={UC-ID}-SC{N}
# @trace.source=<official .feature path>
# @trace.test_type=functional|integration|e2e|non-functional
```

> `/qc-run-test` ghi `pass|fail|skip|not_run` + `qc_run_at` vào `{trace_dir}/{UC-ID}.tsv` (song song với `dev_selftest`), surfaced trong Living Docs là kết quả QC automation **chính thức**. Mỗi FAIL được phân loại script-bug (fix selector/logic) vs product-gap (giữ FAIL + evidence, không bao giờ fake-pass).

---

*Xem thêm:* [Trace TSV Schema](trace-schema.md) (`qc_status` vs `dev_selftest`) · [Command Reference](commands.md) (pipeline `/qc-*`) · [02 · Guides · QC Automation](../02-guides/qc-automation.md).
