---
doc_id: ADR-0002
title: Product, Listing and Shop Model
status: superseded
owner: Cartasy Product & Engineering
last_updated: 2026-10-19
version: 2.0.0
source_of_truth: Lịch sử quyết định product/listing/shop; superseded bởi ADR-0009
review_cycle: Khi mở seller hoặc đổi catalog model
---

# ADR-0002: Product/variant tách listing/shop

## Context

Marketplace có thể có nhiều shop bán cùng sản phẩm với giá, tồn kho và trạng thái riêng. Đặt giá trực tiếp trên product sẽ làm trùng nội dung hoặc mất ranh giới shop.

## Decision

Product/variant là canonical content; listing là offer của shop cho một variant, sở hữu giá VND, publish state và stock reference. Cart/order tham chiếu listing; order item snapshot nội dung và giá. Shop ID có mặt từ F1 dù chỉ có platform shop. Seller tạo proposal/mapping qua moderation, không tự sửa canonical product của shop khác.

Bản triển khai một platform shop ở F1 đã được thay bởi [ADR-0009](ADR-0009-multiseller-mvp.md). Model product/listing/shop vẫn được kế thừa.

## Consequences

Có thêm bảng/flow moderation nhưng mở nhiều shop mà không đổi checkout. Listing/stock được kiểm tra shop scope ở server; product merge không sửa lịch sử đơn. Không dùng affiliate offer/link làm checkout route.
