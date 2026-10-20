---
name: dev-worker
description: "Xây background processing trong apps/worker: outbox, job, mock settlement, notification và retry. Không dùng cho HTTP/UI feature thuần túy."
---

# Dev Worker

- Đọc `docs/apps/worker/README.md`, domain spec của event/job và `docs/platform/deployment-operations.md`.
- Worker gọi public use case trong `packages/server`; SQL thuộc adapter `packages/database`. Không tạo business rule hay state transition riêng trong handler.
- Dùng PostgreSQL outbox/job với claim lease, retry/backoff, inbox dedupe và handler idempotent. Gọi SMTP/Push/S3 ngoài transaction; hỗ trợ crash/replay và dead-letter.
- Payment, shipping, refund của Cartasy chỉ mô phỏng. Notification phải theo consent, locale và phạm vi người nhận; không chứa PII nhạy cảm trong event/push. Kiểm duplicate, retry, deadline bằng PostgreSQL thật và fake clock.
