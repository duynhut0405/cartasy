---
doc_id: DOMAIN-REVIEWS
title: Review và moderation
status: accepted
owner: Trust & Reviews
last_updated: 2026-10-19
version: 4.2.0
source_of_truth: Điều kiện đánh giá, phản hồi seller và chống lạm dụng
---

# Đánh giá trải nghiệm mô phỏng

## Phạm vi và quyền

Review F1 mô tả **trải nghiệm mua sắm và quy trình giao mô phỏng**, không chứng thực người dùng đã nhận/sử dụng hàng thật. Web, email và API luôn có `simulation: true` và nhãn tương ứng; không dùng badge “đã mua hàng thật”. Buyer chỉ review order item của mình sau `delivered_simulated`. Seller có thể phản hồi review thuộc shop mình với capability `reviews.reply`; không thể duyệt, ẩn, xóa hoặc sửa rating. Admin `reviews.moderate` xử lý queue/report theo reason và audit. Public chỉ đọc review `published`.

## Luồng và invariant

Buyer tạo tối đa một review cho mỗi order item trong 30 ngày sau giao mô phỏng (mặc định kỹ thuật cần Product duyệt theo [audit](../delivery/audit-and-open-decisions.md)): rating 1–5, title tối đa 120 ký tự, body 10–2000 ký tự, tối đa năm ảnh đã scan. Unique `(buyer_id, order_item_id)` chặn retry/concurrency; tạo lại cùng `Idempotency-Key` trả review cũ, khác payload trả conflict. `pending_moderation → published/rejected`; `published → hidden` khi report hợp lệ; sửa review đã publish trong bảy ngày chuyển lại `pending_moderation`, giữ revision lịch sử. “Xóa” của buyer là soft delete/yêu cầu ẩn theo retention. Review bị refund vẫn tồn tại với trạng thái hoàn tiền mô phỏng, không bị ẩn vì điểm thấp. Seller reply tối đa một active reply/review, được sửa theo revision và qua moderation; không chứa PII/link bán ngoài.

Aggregate rating chỉ tính `published`, cập nhật bằng event idempotent và có job rebuild/đối soát. Sample count hiển thị thật, không seed review giả lên public. Product và shop rating tách riêng; cùng item không được nhân bản theo listing.

## Hợp đồng và lỗi

Buyer `POST /v1/orders/:id/items/:itemId/review`, `GET /v1/me/reviews`, `PATCH /v1/me/reviews/:id`; public `GET /v1/products/:slug/reviews`; seller `GET /v1/seller/shops/:shopId/reviews` và `POST /v1/seller/shops/:shopId/reviews/:id/reply`; admin `GET /v1/admin/reviews/queue`, `POST /v1/admin/reviews/:id/decision`. Report abuse dùng `POST /v1/reports` với `resourceType=review` (authenticated, rate limited); admin action có `If-Match`/reason. [API conventions](../system/api-contracts.md) sở hữu shape chung.

`404` cho cross-owner/cross-shop, `409 REVIEW_NOT_ELIGIBLE` cho chưa giao/hết cửa sổ/trùng item, `409 VERSION_CONFLICT` cho revision cũ, `422` cho content/media, `429` cho spam. Media private qua upload intent, scan trước moderation; render escaped/sanitized. Outbox `review.submitted/published/reported` cập nhật notification/aggregate sau commit. Bảng `reviews, review_media, review_replies` và `abuse_reports` do Risk sở hữu và unique/index tại [data model](../system/data-model.md).

## Kiểm thử và UI

Web hiển thị eligibility, thời hạn, preview nhãn mô phỏng, pending/rejected reason và report. Khu Seller `/seller/*` trong Admin workspace có review shop/reply; khu Platform `/platform/*` có queue/decision/report theo capability. Test cross-shop, retry/concurrency, edit window, refund, reply moderation, XSS/media, aggregate rebuild và không hiện review chưa publish.
