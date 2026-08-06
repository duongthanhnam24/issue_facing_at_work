# Index

Danh sách nhanh mọi issue đã ghi. Định dạng: `- [Tiêu đề](đường-dẫn) — 1 dòng mô tả ngắn`

## Code

- [Nx monorepo: TS báo missing export sau rebase (stale .d.ts)](code/2026-07-18-nx-stale-dts-sau-rebase.md) — Fix bằng `pnpm nx build <lib>` để emit lại `.d.ts` sync với source mới
- [NestJS `UnknownDependenciesException` — provider chưa export khỏi module chứa nó](code/2026-08-06-nestjs-unknown-dependencies-export-provider.md) — `providers` = private, `exports` = public. Cần export + import đầy đủ

## Database

- [Prisma client không nhận model mới sau khi merge / sửa schema](database/2026-07-18-prisma-client-khong-nhan-model-moi.md) — Phải chạy `pnpm prisma generate` để regenerate client

## DevOps

- [GitLab CI: `needs:` trỏ tới job bị `when: never` → pipeline không tạo được](devops/2026-07-18-gitlab-needs-when-never-conflict.md) — `when: never` = xoá job khỏi pipeline, không phải skip. Fix: bỏ `needs:` khi job cha không bao giờ chạy cùng job con

## Architecture

- [Fair-per-user render scheduler — Hybrid Redis counter + DB status](architecture/2026-08-06-fair-per-user-render-scheduler.md) — Chống queue monopoly khi 1 user batch 20 video. Cap K per user (Redis Lua atomic) + buffer durable (DB `PENDING_DISPATCH`) + trigger event-driven, không cần BullMQ Pro

## Security

- [Webhook & Cơ chế ký HMAC](security/2026-07-16-webhook-hmac-signature.md) — Thiết kế webhook system TTS với HMAC-SHA256 chống spoofing, tampering, replay attack
