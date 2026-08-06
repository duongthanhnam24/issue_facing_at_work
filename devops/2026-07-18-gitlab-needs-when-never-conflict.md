# GitLab CI: `needs:` trỏ tới job bị `when: never` → pipeline không tạo được

**Ngày:** 2026-07-18
**Category:** devops
**Tags:** gitlab-ci, pipeline, yaml
**Stack:** GitLab CI/CD

---

## Context

`.gitlab-ci.yml` có template `.rules_skip_prod` set `when: never` cho 3 job verify (`lint`, `typecheck`, `di-boot`) trên nhánh `main` và `vi-ai`.
`docker-build` chỉ chạy trên `main`/`vi-ai` và khai báo `needs: [lint, typecheck, di-boot]`.

Ý định ban đầu: trên prod bỏ qua verify, chạy thẳng `docker-build` → deploy.

## Symptom

Push vào `main` → pipeline **không được tạo**, GitLab báo YAML error dạng:
`"docker-build: needs 'lint' but this job is not in the pipeline"`.
Không có gì chạy, không deploy được.

## Root cause

GitLab tạo pipeline theo **2 phase**:

**Phase 1 — Resolve rules:** duyệt `rules:` của từng job, quyết định job nào **có mặt** trong pipeline. `when: never` = job **bị xoá khỏi pipeline**, không phải "skip qua".

**Phase 2 — Validate DAG:** kiểm tra mọi `needs:` phải trỏ tới job **cũng có mặt** trong pipeline. Trỏ tới job đã bị loại ở Phase 1 → pipeline reject.

Áp vào case trên `main`:

| Job | Rule khớp | Kết quả Phase 1 |
|---|---|---|
| `lint`, `typecheck`, `di-boot` | `.rules_skip_prod` → `when: never` | **Loại khỏi pipeline** |
| `docker-build` | rule main → `on_success` | Có mặt, cần `lint`... |
| `deploy-prod-main` | rule main → `manual` | Có mặt |

Phase 2: `docker-build.needs` trỏ tới 3 job không tồn tại → gãy pipeline.

## Fix

**Sai lầm trung gian:** thêm `optional: true` vào `needs:`:

```yaml
needs:
  - job: lint
    optional: true
  # ...
```

`optional: true` nói: "có thì chờ, không có thì bỏ qua". Pipeline chạy được ngay.

**Nhưng:** phân tích lại, kịch bản "docker-build có mặt + 3 job kia cũng có mặt" **không bao giờ xảy ra** với config hiện tại (docker-build chỉ chạy trên main/vi-ai, mà trên đó thì 3 job bị `never`). Nên `needs:` giờ là **code chết** — abstraction thừa cho tương lai không chắc có.

**Fix cuối:** bỏ hẳn `needs:` khỏi `docker-build`:

```yaml
docker-build:
  image: docker:27-cli
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "vi-ai"'
  # KHÔNG có needs:
  variables:
    DOCKER_BUILDKIT: '1'
```

## Lesson

- **`when: never` ≠ "skip"** — job **biến mất** khỏi pipeline. `needs:` trỏ tới nó → pipeline không tạo được.
- **Phân biệt 2 khái niệm dễ nhầm trong GitLab:**
  - `when: never` = job **không tồn tại** trong pipeline
  - Job tồn tại nhưng bị skip vì dependency fail = job **có tồn tại**, chỉ không chạy → `needs:` vẫn OK
- **`optional: true`** = phao cứu sinh cho `needs:`, chỉ có tác dụng khi kịch bản "cha có mặt + con vắng mặt" thật sự xảy ra. Nếu kịch bản đó không tồn tại → là code chết.
- **Đừng giữ abstraction cho "tương lai không chắc"** — bỏ `needs:` gọn hơn `optional: true` khi 3 job kia không bao giờ có mặt cùng lúc với `docker-build`.
- **Nhớ giá trị `when:` khác nhau:**
  - `on_success` (mặc định) — có mặt + chờ xanh
  - `manual` — có mặt + chờ người bấm
  - `always` — có mặt + chạy bất kể
  - `never` — **xoá job** (khác với "skip")
- **Verify trước khi push:** dán YAML vào **CI/CD → Editor → Lint** và simulate với `CI_COMMIT_BRANCH=main`. Bug này sẽ lộ ngay, không cần push thử.

## Reference

- [GitLab — `rules:when`](https://docs.gitlab.com/ci/yaml/#ruleswhen)
- [GitLab — `needs:optional`](https://docs.gitlab.com/ci/yaml/#needsoptional)
