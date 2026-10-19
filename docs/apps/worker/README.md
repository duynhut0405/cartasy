---
doc_id: APP-WORKER
title: Worker và background processing
status: accepted
owner: Backend Operations
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Setup, src, job/event contract và vận hành apps/worker
---

# Worker

[Preview thư viện, module và state/store của toàn workspace](../source-preview.md).

## Vai trò và stack

`apps/worker` là Node 24/TypeScript 6 ESM process, Nest application context (không HTTP public) để dùng DI/use cases chung. Thư viện trực tiếp: `@nestjs/core`, `@nestjs/common`, `@cartasy/server`, `@cartasy/database`, `@cartasy/config`, `pino`; `nodemailer`, `web-push@3.6.7` và S3 SDK nằm trong adapter owner. PostgreSQL outbox polling + `SKIP LOCKED` là queue baseline, không thêm Redis/BullMQ. Worker chỉ thực hiện background effect, không sở hữu business rule khác API.

## `src` chuẩn

```text
src/
  bootstrap/
    main.ts
    worker.module.ts
    providers.ts
  outbox/
    poller.ts
    dispatcher.ts
    inbox-dedup.ts
    event-registry.ts
  handlers/
    payments/
      settle-mock-payment.ts
      expire-payment.ts
    notifications/
      create-inbox-item.ts
      send-email.ts
      send-web-push.ts
      fanout-campaign-batch.ts
      resolve-recipients.ts
      resolve-template-locale.ts    # recipient preference snapshot + version
    fulfillment/
      advance-shipment.ts
    media/
      process-image.ts
    discovery/
      refresh-search-projection.ts
    analytics/
      project-event.ts
  jobs/
    poller.ts
    lease.ts
    checkpoint.ts
  scheduler/
    start-scheduled-campaigns.ts
    expire-reservations.ts
    expire-voucher-holds.ts
    remind-support-sla.ts
    reconcile-commerce.ts
  health/
    heartbeat.ts
    metrics.ts
```

Campaign batch job fanout tối đa 500 recipient/keyset/checkpoint, kiểm consent/preference/shop eligibility trước từng delivery; unique campaign+recipient+channel, frequency cap/quiet hours, pause/kill switch. Web Push lỗi 404/410 revoke subscription, 429/5xx retry; push payload tối thiểu, không PII. Handler nhận event versioned, validate schema, gọi public application use case và ghi inbox dedupe. Không SQL tùy tiện ngoài adapter `database`; không tự sửa order/payment state. Event payload tối thiểu gồm `eventId, type, version, aggregateId, occurredAt, correlationId`; không chứa raw OTP, email reset URL, địa chỉ đầy đủ. Template email lấy dữ liệu an toàn qua port và snapshot cần thiết. Handler idempotent vì delivery at least once.

## Poll, retry và scheduler

Unique delivery intent/inbox bảo đảm kết quả logic không nhân đôi; SMTP/Web Push bên ngoài vẫn có thể phát trùng hiếm gặp sau provider accept rồi worker crash trước khi lưu receipt. Không ghi SLO “external exactly-once”; metric và runbook theo dõi duplicate/retry.

Poll batch 100 event, claim bằng `FOR UPDATE SKIP LOCKED` với lease/heartbeat; concurrency theo handler, giữ thứ tự per aggregate nếu phụ thuộc. Transaction claim ngắn; gọi SMTP/Web Push/S3 **ngoài** transaction. Success mark delivered; transient failure exponential backoff+jitter, tối đa 8 lần/24h; permanent/schema-unknown vào dead-letter và alert. Graceful shutdown dừng claim, hoàn tất job đang chạy trong 30 giây rồi nhả lease để retry. Reprocess phải có command có auth/audit, không chỉnh DB thủ công.

Scheduler dùng injectable clock và distributed lock/advisory lock để nhiều replica không xử lý cùng deadline; operation vẫn idempotent. Payment pending hết hạn 15 phút theo [payment](../../domains/payment-simulation.md); reservation và voucher hold giải phóng sau settlement failure/cancel, không tự release khi payment vừa paid. Shipment timeline chỉ bắt đầu sau paid; notification sau commit; return refund qua owner use case. Reconciliation so sánh order total/shop totals, stock/reservation, payment/refund và outbox lag, alert sai lệch, không tự sửa khoản tiền.

## Env, setup và kiểm thử

`APP_ENV`, `DATABASE_URL`, `SMTP_HOST`/`SMTP_PORT`/`EMAIL_FROM`, `S3_*`, `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT`, `MOCK_PAYMENT_MODE=enabled`, `LOG_LEVEL`, `WORKER_CONCURRENCY=4`, `OUTBOX_POLL_MS=1000`. Parse fail-fast; worker không dùng browser public env. Production có một secret set theo process, không share credentials quá quyền.

1. Cài workspace và chạy migration/seed/dependencies theo [local development](../../platform/local-development.md); bật API trước để tạo outbox event.
2. Tạo `@cartasy/worker` scripts `dev/build/start/lint/typecheck/test`, Nest application context và cây `src`; inject adapter/clock/ID generator.
3. Chạy `pnpm --filter @cartasy/worker dev`; tạo checkout qua API, xác nhận payment settle, shipment/notification, email trong Mailpit, push test subscription, campaign product/system announcement và heartbeat/outbox/job lag.
4. Gate: unit state machine, integration thật với PostgreSQL (duplicate, crash/retry, lease, dead-letter), test fake clock cho deadline và smoke shutdown. Runbook vận hành ở [operations](../../platform/deployment-operations.md).

Lỗi handler không được làm crash toàn poller; nhưng readiness/alert phải báo backlog và dead-letter. Không đánh dấu đơn paid chỉ vì email gửi thành công.

F3 partner report import và ad-billing reconciliation (nếu triển khai) là job/handler của owner riêng, có event version/receipt/dedupe và secret tách worker mock payment. Không nhận click/paid_simulated làm bằng chứng partner commission hay ad revenue, không ghi trực tiếp bảng `orders/payments`. [Affiliate](../../domains/affiliate.md), [Ad billing](../../domains/advertising-billing.md).
