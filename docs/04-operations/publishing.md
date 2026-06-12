[📚 Docs](../README.md) › [Operations](README.md) › Publishing

# Publishing — Publish lên npm

> Hướng dẫn publish package `@anhth2/spec-driven-dev-plugin` lên npm: setup lần đầu, publish version mới, kiểm tra sau publish, và transfer ownership.

Package: `@anhth2/spec-driven-dev-plugin`
Registry: <https://www.npmjs.com/package/@anhth2/spec-driven-dev-plugin>

---

## Mục lục

1. [Yêu cầu](#1-yêu-cầu)
2. [Setup lần đầu](#2-setup-lần-đầu)
3. [Publish version mới](#3-publish-version-mới)
4. [Kiểm tra sau khi publish](#4-kiểm-tra-sau-khi-publish)
5. [Transfer package sang account khác](#5-transfer-package-sang-account-khác)

---

## 1. Yêu cầu

- Node.js đã cài (`node -v`)
- Tài khoản npm: <https://www.npmjs.com> (hiện đang dùng account `anhth2`)

---

## 2. Setup lần đầu

### 2.1 — Đăng nhập npm

```powershell
npm login
```

Trình duyệt sẽ mở trang xác thực npm. Đăng nhập bằng account `anhth2`, sau đó quay lại terminal. Kiểm tra:

```powershell
npm whoami
# output: anhth2
```

### 2.2 — Tạo Access Token (nếu cần dùng CI hoặc tránh 2FA mỗi lần)

1. Vào <https://www.npmjs.com> → Avatar → **Access Tokens**
2. **Generate New Token** → **Granular Access Token**
3. Điền:
   - Name: `publish-spec-driven-dev`
   - Expiration: 1 year
   - Packages: chọn `@anhth2/spec-driven-dev-plugin` → **Read and write**
4. Copy token, lưu vào nơi an toàn

Publish với token:

```powershell
npm publish --token <your-token>
```

---

## 3. Publish version mới

### Bước 1 — Cập nhật nội dung commands (nếu có thay đổi)

Sửa các file trong thư mục `commands/`.

### Bước 2 — Tăng version trong `package.json` (semver)

| Loại thay đổi | Lệnh | Ví dụ |
|---|---|---|
| Fix nhỏ, sửa lỗi | `npm version patch` | 0.1.0 → 0.1.1 |
| Thêm command mới | `npm version minor` | 0.1.0 → 0.2.0 |
| Thay đổi lớn, breaking | `npm version major` | 0.1.0 → 1.0.0 |

```powershell
# Ví dụ: thêm command mới
npm version minor
```

Lệnh này tự động cập nhật `version` trong `package.json` và tạo git commit + tag.

### Bước 3 — Publish

```powershell
npm publish
```

### Bước 4 — Push git (bao gồm tag vừa tạo)

```powershell
git push && git push --tags
```

---

## 4. Kiểm tra sau khi publish

```powershell
# Xem version mới trên npm
npm view @anhth2/spec-driven-dev-plugin version

# Chạy thử
npx @anhth2/spec-driven-dev-plugin@latest
```

---

## 5. Transfer package sang account khác

Khi cần chuyển quyền sở hữu package cho người khác (ví dụ: sang account `edupia-team`):

### Thêm maintainer

```powershell
npm owner add <npm-username> @anhth2/spec-driven-dev-plugin
```

### Transfer toàn bộ

```powershell
npm access grant read-write <npm-username> @anhth2/spec-driven-dev-plugin
```

Hoặc làm thủ công trên website:

1. Vào <https://www.npmjs.com/package/@anhth2/spec-driven-dev-plugin>
2. Tab **Settings** → **Maintainers** → thêm username mới

> **Lưu ý:** Nếu muốn đổi tên package, cần publish lại với tên mới vì npm không cho đổi tên package đã publish. Sau đó deprecate package cũ:
> ```powershell
> npm deprecate @anhth2/spec-driven-dev-plugin "Moved to <new-package-name>"
> ```

---

*Xem thêm:* [Sync & Update](sync-and-update.md) (`/update-framework` kéo version mới từ npm về project) · [Bug Flow](bug-flow.md).
