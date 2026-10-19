---
doc_id: ADR-0003
title: Simulated Payment and Fulfillment
status: accepted
owner: Cartasy Product & Engineering
last_updated: 2026-10-19
version: 2.0.0
source_of_truth: Ranh giới payment/order/fulfillment mô phỏng
review_cycle: Trước khi đề xuất tiền hoặc giao nhận thật
---

# ADR-0003: Mock payment và giao nhận

## Context

Sản phẩm cần luồng mua sắm như marketplace nhưng không chuyển tiền hoặc giao hàng. Một fake success đồng bộ thiếu failure/timeout/replay và làm khó quản lý order thực tế.

## Decision

Checkout tạo order/payment pending và reservation trong transaction. Sau commit, MockPaymentGateway nội bộ tạo success/failure/timeout; settlement use case idempotent cập nhật payment/order/stock và outbox. Shipment/timeline là mô phỏng, không có carrier thật. UI/email/notification ghi rõ không thu tiền/giao hàng. Không thu card/OTP thật. Scenario override chỉ ở local/test, không mở public.

## Consequences

Có thể test hành trình hoàn chỉnh, concurrency và notification giống hệ thống thương mại mà không tạo giao dịch tài chính. Cần reconciliation cho pending/out-of-order event. Thanh toán/giao nhận thật là thay đổi phạm vi cần đặc tả, legal/security/operations review và ADR mới, không chỉ đổi adapter.
