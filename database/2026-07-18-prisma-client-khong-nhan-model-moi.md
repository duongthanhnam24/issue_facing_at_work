# Prisma client không nhận model mới sau khi merge / sửa schema

**Ngày:** 2026-07-18
**Category:** database
**Tags:** prisma, workflow, orm
**Stack:** Node.js, Prisma, pnpm

---

## Context

Sau mỗi lần **merge code** hoặc **thêm/sửa model** trong `schema.prisma`, Prisma client cũ vẫn còn — code TypeScript báo lỗi không tìm thấy model mới, hoặc field mới không autocomplete.

## Symptom

- Type error: `Property 'newModel' does not exist on type 'PrismaClient'`
- Runtime error khi query model vừa thêm
- IDE không autocomplete field mới

## Root cause

Prisma client là **code được generate** (nằm trong `node_modules/@prisma/client`), không tự động sync với `schema.prisma`. Khi schema thay đổi (do merge từ branch khác hoặc mình thêm model), phải chạy lệnh generate để tạo lại client.

## Fix

```bash
pnpm prisma generate
```

## Lesson

- **Sau mỗi lần `git pull` / `git merge`** có thay đổi `schema.prisma` → chạy `pnpm prisma generate`.
- **Sau khi thêm/sửa model** trong schema → chạy `pnpm prisma generate`.
- Cân nhắc thêm vào **post-merge hook** hoặc **postinstall script** trong `package.json` để tự động:
  ```json
  "scripts": {
    "postinstall": "prisma generate"
  }
  ```
