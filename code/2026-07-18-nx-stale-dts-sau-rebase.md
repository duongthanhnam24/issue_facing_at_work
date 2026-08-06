# Nx monorepo: TS báo missing export sau rebase (stale .d.ts)

**Ngày:** 2026-07-18
**Category:** code
**Tags:** nx, typescript, monorepo, workflow, pnpm
**Stack:** Nx, TypeScript project references, pnpm, NestJS

---

## Context

Repo `api-mkt-post-video` — Nx monorepo dùng TypeScript project references giữa các lib. Sau khi **rebase** kéo về code mới của lib `infra-create-video`, chạy `tsc --build tsconfig.lib.json` ở `modules-connections` báo lỗi.

## Symptom

```
src/application/services/template-preview.service.ts:8:3 - error TS2305:
  Module '"@nestjs-fastify-nx/infra-create-video"' has no exported member 'buildNewsPanel'.

src/application/services/template-preview.service.ts:9:8 - error TS2305:
  Module '"@nestjs-fastify-nx/infra-create-video"' has no exported member 'PanelBox'.
```

Nhưng check barrel `libs/infra/create-video/src/index.ts` thì **đã export đúng** `buildNewsPanel`, `PanelBox`.

## Investigation

- Đã grep xác nhận barrel export tồn tại → không phải lỗi source code
- Thử `pnpm nx reset` → fail: `EBUSY: resource busy or locked` (Nx daemon đang giữ file `.nx/workspace-data/*.db` trên Windows)
- Phải `pnpm nx daemon --stop` trước

## Root cause

**Stale `.d.ts` artifacts sau rebase.**

- Rebase kéo code mới của `infra-create-video` (có `buildNewsPanel`, `PanelBox`)
- Nhưng `dist/*.d.ts` cũ (build trước rebase) **chưa có** 2 export đó
- TS project references của `modules-connections` chỉ vào `.d.ts` cũ → báo "no exported member"

## Fix

Rebuild lib bị ảnh hưởng để emit lại `.d.ts` mới:

```bash
pnpm nx build infra-create-video
```

Nếu Nx cache stubborn, thêm `--skip-nx-cache`:

```bash
pnpm nx build infra-create-video --skip-nx-cache
```

Nếu `nx reset` fail vì file lock trên Windows:

```bash
pnpm nx daemon --stop
pnpm nx reset
```

## Lesson

- **Sau mỗi lần rebase/pull có thay đổi lib được consume**, phải rebuild lib đó để `.d.ts` sync với source.
- Trên **Windows** thường gặp `EBUSY` khi `nx reset` — luôn `nx daemon --stop` trước.
- Mẹo cho lần sau, rebase/pull xong chạy 1 phát cho gọn:
  ```bash
  # Build tất cả lib affected
  pnpm nx affected -t build

  # Hoặc theo scope tag
  pnpm nx run-many -t build -p tag:scope:infra --skip-nx-cache
  ```
- Cân nhắc thêm vào **post-merge / post-rewrite git hook** để auto rebuild affected libs.

## Reference

- [Nx — TypeScript project references](https://nx.dev/concepts/decisions/project-dependency-rules)
- Liên quan: [[2026-07-18-prisma-client-khong-nhan-model-moi]] — cùng dạng "stale generated artifact sau merge/rebase"
