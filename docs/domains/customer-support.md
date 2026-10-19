---
doc_id: DOMAIN-SUPPORT
title: Customer support
status: accepted
owner: Customer Service
last_updated: 2026-10-19
version: 4.2.0
source_of_truth: Ticket, hội thoại, phân quyền và SLA
---

# Hỗ trợ khách hàng

## Scope và quyền

Buyer tạo ticket từ tài khoản, order/shop order/return thuộc mình. Seller với `support.reply` chỉ đọc/reply ticket được gắn shop của mình; API redacts PII không cần thiết, internal note và ticket shop khác. Admin support có quyền assign, reply, close/escalate; quyền sửa order/payment nằm ở owner use case và cần reason/audit riêng. Không quảng bá F1 là live chat: ticket là async, notification báo câu trả lời. Không nhận số thẻ, OTP ngân hàng hoặc bằng chứng thanh toán thật.

## Dữ liệu và trạng thái

Loại ticket `account/catalog/order/payment_simulated/shipping_simulated/return/review/shop/other`; subject 5–120, message 10–5000 ký tự, tối đa năm attachment ≤10 MB/file đã scan, URL private ngắn hạn. Mỗi ticket có buyer owner, optional shop/order/return reference, assignee, priority và version. Liên kết order phải thuộc buyer; nếu gắn shop thì shop phải thuộc order đó. Message append-only, actor role và visibility `customer_visible/internal`; internal note chỉ admin. Escalation bảo toàn lịch sử.

`open → in_progress → waiting_customer → resolved → closed`; reopen từ resolved/closed trong bảy ngày, tạo history/version mới. Buyer close ticket của mình sau resolved; admin có thể close với reason. Seller reply không đổi state sang resolved nếu policy không cấp quyền. Concurrent reply dùng idempotency key và sequence; state mutation dùng `If-Match`. Ticket shop suspended vẫn cho xử lý order cũ qua quyền support được platform cho phép, không mở catalog mutation.

Mặc định kỹ thuật cho SLA: giờ làm việc 09:00–18:00 thứ Hai–Sáu Asia/Ho_Chi_Minh, first response 24 giờ làm việc, resolution 72 giờ làm việc; lịch nghỉ versioned và scheduler nhắc/escalate, không tự resolve. **SLA này là giả định cần duyệt trước public launch**, theo [audit](../delivery/audit-and-open-decisions.md). Chỉ công bố SLA với khách sau khi chủ dự án xác nhận.

## API, lỗi và event

Buyer `GET/POST /v1/me/support/tickets`, `GET /v1/me/support/tickets/:id`, `POST /v1/me/support/tickets/:id/messages`, `POST /v1/me/support/tickets/:id/close|reopen`. Seller `GET /v1/seller/shops/:shopId/support/tickets` và `POST /v1/seller/shops/:shopId/support/tickets/:id/messages`. Admin `GET /v1/admin/support/tickets`, `POST /v1/admin/support/tickets/:id/assign|transition|messages`. [API contracts](../system/api-contracts.md) chuẩn hóa envelope; [data model](../system/data-model.md) sở hữu ticket/message/attachment.

Cross-owner/shop `404`; closed/invalid transition hoặc stale version `409`; content/reference/file `422`; spam `429`. Outbox `support.created/replied/state_changed/sla_breached` chỉ phát sau commit, recipient lọc theo visibility. Attachment không nằm trong email/push; email chỉ có deep link. Không log message/attachment URL/token.

## UI và kiểm thử

Web có danh sách/detail/timeline, loading/error, trạng thái chờ; khu Seller `/seller/*` trong Admin workspace có queue shop scoped; khu Platform `/platform/*` có queue theo SLA/priority/assignee, internal note rõ khác reply. Test cross-tenant, redaction, internal note leak, XSS/file scan, concurrent reply, reopen window, holiday/DST logic, scheduler retry và notification dedupe.
