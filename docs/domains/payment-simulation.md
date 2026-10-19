---
doc_id: DOMAIN-PAYMENT
title: Payment simulation
status: accepted
owner: Payments
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Payment state, attempt và settlement giả lập
---

# Thanh toán mô phỏng

Mọi phương thức `mock_card`, `mock_e_wallet`, `mock_bank` chỉ là lựa chọn giao diện; không yêu cầu số thẻ, tài khoản ngân hàng hoặc OTP ngân hàng. Không có payment provider/callback thật. Checkout tạo `payments.pending` với amount bằng order total và một attempt. Mock adapter nhận kịch bản từ cấu hình server/test harness bảo vệ: success, failure, timeout. Browser staging/production không được chọn kết quả. UI ở tất cả bước có nhãn “Không thu tiền thật”.

## State machine

`pending → paid_simulated` khi success; `pending → failed` khi failure; timeout giữ `pending` và scheduler đối soát/giải quyết sau deadline. Không có transition từ `failed` sang paid; buyer muốn thử lại phải tạo order mới từ cart mới. `paid_simulated → partially_refunded_simulated → refunded_simulated` theo refund của shop order; amount refunded không vượt paid. `cancelled` chỉ cho pending bị hủy trước kết luận và giải phóng reservation/voucher. Payment attempt lưu scenario, request key, event ID, timestamp, result; không lưu dữ liệu tài chính thật.

Settlement khóa payment/order/reservation và CAS state. Success: commit reservation, commit voucher, xác nhận order/shop orders, tạo shipment mô phỏng và outbox. Failure: release reservation/voucher, chuyển order payment_failed và outbox. Duplicate callback/job không tạo side effect lặp. Timeout: sau 15 phút worker chuyển sang failure theo policy F1 nếu vẫn pending; test harness có thể giữ pending để kiểm thử retry. API `GET /v1/orders/:id/payment` là read-only cho owner.

Refund mô phỏng phát sinh từ cancel/return, idempotency theo request key; lưu amount/currency/order allocation và lý do. Không phát thông điệp “tiền đã về tài khoản”; wording “hoàn tiền mô phỏng đã hoàn tất”. Audit mọi transition với actor/source và version. Lỗi `INVALID_STATE_TRANSITION`, `PAYMENT_PENDING`, `CONFLICT`; API không có route công khai quyết định success/failure. Test: duplicate/delayed settlement, payment success sau timeout, partial refund nhiều shop, race cancel, no real payment fields/SDK. Quy tắc tổng thể ở [ADR-0003](../decisions/ADR-0003-simulated-payment.md).


## Attempt, timeout và error contract

| Tình huống | Payment/order/stock | Client và thông báo |
|---|---|---|
| Success | `paid_simulated`/confirmed, reservation+voucher committed | Result có nhãn mô phỏng, inbox buyer/seller đúng scope |
| Failure | `failed`/payment_failed, hold released | Buyer có thể tạo giỏ/đơn **mới** từ item eligible |
| Timeout ngắn | `pending` giữ order/key, không release | UI pending/poll/SSE; không retry key mới |
| Hết hạn 15 phút | Nếu vẫn pending, fail idempotent và release | Event failure/notification một lần |
| Duplicate/delayed outcome | CAS + attempt/event unique; terminal không đảo | Trả kết quả đã lưu, alert nếu provider giả lập mâu thuẫn |

Không có payment webhook public trong MVP; test harness ép kịch bản chỉ local/test, fail closed staging/production. `MockPaymentGateway` là port thuần, không được dùng request browser để chọn outcome. Refund mô phỏng theo shop-order/return snapshot và `refunded_minor <= paid_minor`; không có card/bank/OTP/payout. Adapter webhook tương lai ở [Integrations](../platform/external-integrations.md). Test race settlement vs cancel/expiry, duplicate attempt, amount mismatch, refund nhiều shop và bảo đảm không gọi external payment host.
