---
doc_id: ADR-0009
title: Multi-seller Marketplace in MVP
status: accepted
owner: Cartasy Product & Engineering
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Product/listing/shop model và multi-seller rollout từ MVP
supersedes: ADR-0002
review_cycle: Khi đổi tenant hoặc catalog/checkout ownership
---

# ADR-0009: Marketplace đa seller từ MVP

## Context

Yêu cầu sản phẩm đã xác nhận buyer có thể mua từ nhiều seller ngay MVP, seller vận hành shop trong `apps/admin`. Thiết kế trước đây ở ADR-0002 giữ được model product/listing/shop nhưng giả định chỉ một platform shop ở F1. Giả định triển khai đó không còn đúng.

## Decision

Từ F1 có nhiều shop active và nhiều seller độc lập. `Product/Variant` là canonical content; `Listing` là offer của một shop cho một variant, sở hữu giá VND, publish state và stock. Seller đề xuất product/variant mới hoặc mapping tới canonical product qua moderation; không sửa nội dung canonical của shop khác. Cart chọn listing; checkout tạo một parent order và một shop order mỗi shop, snapshot item/giá/discount/phí bất biến. Stock, promotion, order, support, report, notification và authorization đều shop scoped ngay F1. Seller và Platform Admin dùng chung `apps/admin` theo [ADR-0011](ADR-0011-unified-admin-seller-workspace.md). Payment/fulfillment/refund vẫn mô phỏng theo [ADR-0003](ADR-0003-simulated-payment.md).

## Consequences

Foundation cần membership/capability, onboarding, isolation hai shop và hai buyer, moderation, multi-shop checkout/refund và notification recipient đúng shop trước release. Không chờ F2 mới thêm `shop_id` hoặc tách seller UI. F1 tăng phạm vi triển khai; gate ở [roadmap](../delivery/roadmap.md) và [build checklist](../delivery/build-checklist.md) chứng minh đủ quyền, tính đúng đắn và nhãn mô phỏng.
