# Issue Facing At Work

Nơi ghi chép mọi vấn đề, bug, và quyết định kỹ thuật đã đối mặt trong công việc với vai trò Software Engineer.

## Mục đích

- Không lặp lại cùng một lỗi/sai lầm nhiều lần.
- Tra cứu nhanh khi gặp vấn đề tương tự trong tương lai.
- Ghi lại root cause và bài học, không chỉ fix.

## Cấu trúc

```
.
├── README.md              # File này
├── INDEX.md               # Danh sách nhanh mọi issue đã ghi
│
├── code/                  # Bug logic, refactor, framework, thư viện
├── database/              # Query, migration, deadlock, schema
├── devops/                # CI/CD, Docker, K8s, deploy, monitoring
├── architecture/          # Design decision, pattern, trade-off
├── security/              # Auth, vulnerability, secret, permission
│
└── templates/
    └── issue.md           # Template chuẩn cho mỗi issue
```

## Quy ước

### Đặt tên file

```
YYYY-MM-DD-mo-ta-ngan.md
```

Ví dụ: `2026-07-16-deadlock-khi-update-order.md`

### Tag đặc biệt trong tên file

Dùng prefix để đánh dấu loại vấn đề cross-cutting (không phải category riêng):

- `perf-` — vấn đề về performance
- `bug-` — bug production
- `incident-` — sự cố lớn cần postmortem

Ví dụ: `database/2026-07-16-perf-query-users-cham.md`

### Cách ghi

1. Copy `templates/issue.md` sang category phù hợp, đổi tên theo quy ước.
2. Điền đủ các mục trong template.
3. Thêm dòng link vào `INDEX.md`.

## Workflow

1. Gặp vấn đề → ghi ngay khi còn nhớ chi tiết.
2. Sau khi fix → cập nhật phần Root cause và Lesson.
3. Định kỳ (hàng tháng/quý) → đọc lại INDEX.md để nhớ.
