---
doc_id: PLATFORM-INTEGRATIONS
title: External Integrations and Deferred Adapters
status: accepted
owner: Architecture & Security
last_updated: 2026-10-19
version: 4.1.0
source_of_truth: Ranh giới provider, webhook, Web Push/FCM và khả năng tích hợp tương lai
---

# Ranh giới tích hợp

## Đang dùng trong Marketplace MVP

| Tích hợp | Adapter/contract | Quy tắc |
|---|---|---|
| SMTP | `EmailSender` qua Nodemailer; Mailpit local, provider SMTP staging/production | Template version, retry/dedupe, không gửi email nghiệp vụ trong DB transaction |
| S3-compatible storage | `ObjectStore` qua AWS SDK | Bucket private, signed URL ngắn hạn, scan/quarantine, ownership |
| Browser Web Push | Push API + Service Worker ở Web; worker dùng `web-push`/VAPID | Chỉ subscription opt-in; secret VAPID server-side; endpoint encrypted at rest, 404/410 xóa subscription |
| Realtime browser | SSE `/v1/me/stream` cho inbox/order hint | DB vẫn là nguồn chuẩn; reconnect/replay qua cursor, giới hạn connection; polling fallback |
| Mock payment/shipping | `PaymentGateway`/`FulfillmentProvider` nội bộ | Không external charge/carrier; scenario chỉ local/test; event idempotent |

Dùng [Web Push standard](https://developer.mozilla.org/en-US/docs/Web/API/Push_API), [web-push Node library](https://github.com/web-push-libs/web-push), [NestJS SSE](https://docs.nestjs.com/http/server-sent-events). Web Push yêu cầu HTTPS, permission theo thao tác chủ động và Service Worker. Trên iOS, browser push áp dụng cho web app được thêm vào Home Screen theo [Apple Developer](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers); inbox vẫn hoạt động nếu thiết bị/browser không hỗ trợ hoặc từ chối. **FCM không là dependency MVP**: dùng chuẩn Web Push cho web; FCM/APNs chỉ khi có mobile app native hoặc yêu cầu provider cụ thể và ADR mới.

## Realtime và queue: quyết định có điều kiện

F1 ưu tiên PostgreSQL outbox cho domain event và `background_jobs`/`notification_campaign_batches` cho work item, claim bằng `FOR UPDATE SKIP LOCKED`. Event source được commit cùng transaction nghiệp vụ; job chạy at-least-once, inbox/delivery có unique business key. In-process event emitter **không** được dùng làm cơ chế đáng tin cậy. SSE là kênh báo “có dữ liệu mới”, không chuyển toàn bộ PII/state trong stream; client refetch API có quyền. Với nhiều API replica, PostgreSQL LISTEN/NOTIFY chỉ là tín hiệu best-effort; reconnect dùng `Last-Event-ID`/cursor đọc DB nên không mất thông báo. Khi tải/SLO đo được vượt khả năng PostgreSQL, thay adapter bằng Redis Streams/BullMQ hoặc broker có durability, giữ outbox làm nguồn sau transaction và viết ADR/migration/runbook. WebSocket chỉ cần khi có luồng hai chiều thực sự (live chat/live commerce F3), không dùng chỉ để đẩy trạng thái đơn.

## Payment webhook và partner integration sau MVP

**Không có public payment webhook trong MVP.** Nếu bật **ad billing thật** sau này, callback/payment provider route thuộc `ad-billing` namespace tách buyer business API: xác thực signature trên raw body, timestamp/nonce và replay; lưu receipt/event ID unique trước ack; xử lý async/idempotent, reconcile với provider/invoice/ledger. Không map provider success vào `payments.paid_simulated` hoặc Cartasy order. Payment/refund thật cho **đơn mua Cartasy** vẫn chưa được chấp thuận và cần quyết định sản phẩm/ADR khác. Không nhận success do browser gửi. [Ad billing](../domains/advertising-billing.md), [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

## Carrier, FCM, affiliate và third-party feeds sau MVP

- Carrier thật yêu cầu hợp đồng/consent địa chỉ, rate quote, label/tracking, webhook signature, retry/reconcile, retention PII và ADR. Hiện không chuyển địa chỉ cho carrier.
- FCM/APNs cho mobile native cần device token lifecycle, permission, multi-device preference và sender credential; không dùng FCM chỉ để gửi web push nếu chuẩn Web Push đáp ứng.
- Affiliate outbound sau MVP có thể dẫn Buyer sang đối tác mua thật; partner link/redirect, API/feed, report conversion/commission và settlement chỉ dùng khi Cartasy được duyệt vào chương trình cụ thể. Không tạo Cartasy order hoặc suy hoa hồng từ mock payment. Partner API/callback cần credential, quota, signature/receipt, consent, rights, rate limit và đối soát theo hợp đồng; không coi Open Platform/Seller API là quyền Affiliate. [Affiliate](../domains/affiliate.md).
- Seller import/feed chỉ từ nguồn được phép; URL fetch có SSRF protection, quota, staging/preview, provenance và dedupe.

## Contract và failure policy

Mỗi adapter có `timeoutMs`, retry classification, circuit/health metric, idempotency/correlation ID, redacted log và test fake/provider sandbox. External call không ở trong checkout DB transaction. Nếu email/push hỏng, order vẫn đúng và delivery retry/dead-letter; nếu object storage hỏng, media chưa publish. Không tự chuyển payment/ship state chỉ vì notification đã gửi.
