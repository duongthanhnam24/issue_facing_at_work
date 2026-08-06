# NestJS `UnknownDependenciesException` — provider chưa export khỏi module chứa nó

**Ngày:** 2026-08-06
**Category:** code
**Tags:** nestjs, dependency-injection, module
**Stack:** NestJS, TypeScript

---

## Context

`StoryboardPipelineProcessor` (BullMQ processor) inject nhiều dependency, trong đó có `ApplyStoryboardTemplateHandler`. Khi app bootstrap ở `AppModule` thì báo lỗi.

## Symptom

```
[Nest] 1  - 08/06/2026, 9:26:25 AM   ERROR [ExceptionHandler]
UnknownDependenciesException [Error]: Nest can't resolve dependencies of the
StoryboardPipelineProcessor (
  Symbol(VIDEO_POST_REPOSITORY_PORT),
  BullQueue_storyboard-render,
  ExtractArticleHandler,
  GenerateStoryboardHandler,
  ?,                             ← index [4] không resolve được
  UserRenderSlotService,
  Symbol(STORYBOARD_GENERATE_PORT)
).
Please make sure that the argument ApplyStoryboardTemplateHandler at index [4]
is available in the AppModule module.
```

Nest chỉ đích danh: **argument index [4] = `ApplyStoryboardTemplateHandler`** không tìm thấy trong scope của module chứa `StoryboardPipelineProcessor`.

## Root cause

**Module scope trong NestJS DI:**

- Provider register trong `providers: []` chỉ **visible bên trong module đó**.
- Muốn module khác dùng → **phải thêm vào `exports: []`** của module gốc.
- Module tiêu thụ phải **`imports:` module gốc** để lấy được provider đã export.

Trong case này: `ApplyStoryboardTemplateHandler` đang được register ở 1 module (VD `StoryboardHandlersModule`) nhưng **chưa export**, nên module chứa `StoryboardPipelineProcessor` không "thấy" được — dù đã import.

## Fix

Cách bạn làm **đúng** nhưng cần kiểm tra **cả 2 bước** cho đầy đủ:

### Bước 1: Export provider khỏi module chứa nó

```typescript
// storyboard-handlers.module.ts
@Module({
  providers: [
    ApplyStoryboardTemplateHandler,   // ← đã có sẵn
    // ...
  ],
  exports: [
    ApplyStoryboardTemplateHandler,   // ← THÊM VÀO đây
  ],
})
export class StoryboardHandlersModule {}
```

### Bước 2: Đảm bảo module tiêu thụ đã import module gốc

```typescript
// storyboard-pipeline.module.ts (module chứa Processor)
@Module({
  imports: [
    StoryboardHandlersModule,   // ← phải có dòng này
    // ...
  ],
  providers: [
    StoryboardPipelineProcessor,
  ],
})
export class StoryboardPipelineModule {}
```

Nếu chỉ làm bước 1 mà quên bước 2 → vẫn báo cùng lỗi.

## Lesson

- **`providers` = private, `exports` = public.** Provider không tự động visible ra ngoài module.
- **Đọc kỹ error message của Nest** — nó chỉ đúng tên argument và index gây lỗi (dấu `?` trong danh sách). Không cần đoán.
- **Module chứa lỗi trong error message không phải chỗ cần sửa.** Error nói "not available in AppModule" nhưng phải sửa ở **module đang khai báo `ApplyStoryboardTemplateHandler`**, không phải AppModule.
- **Checklist khi thêm provider mới cần dùng ở module khác:**
  1. Register trong `providers: []` của module gốc
  2. Thêm vào `exports: []` của module gốc
  3. `imports:` module gốc vào module tiêu thụ
- Với **CQRS pattern** (nhiều Handler), thường nhóm tất cả Handler thành 1 module riêng và export cả cụm để tiện consume.

## Reference

- [NestJS — Modules (Shared modules)](https://docs.nestjs.com/modules#shared-modules)
- [NestJS — Custom providers (Provider scope)](https://docs.nestjs.com/fundamentals/custom-providers)
