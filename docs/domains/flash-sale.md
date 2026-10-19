---
doc_id: DOMAIN-FLASH-SALE
title: Flash Sale
status: accepted
owner: Promotions & Inventory
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Flash sale F2, quota/stock và pricing concurrency
---

# Flash sale (F2)

Flash sale là campaign có khung giờ và quota riêng trên listing/shop, không phải voucher đổi tên. Chỉ seller của shop active đề xuất listing active/stock hợp lệ; Admin duyệt campaign cấp platform. State `draft → pending_review → scheduled → active → ended`, nhánh `rejected/paused/cancelled`; scheduler kích hoạt theo UTC. Mỗi listing/time window chỉ có một flash sale active; không cho sửa giá/quota sau khi active, chỉ pause/terminate có audit. Giá sale VND integer không âm và thấp hơn giá listing snapshot nếu dùng nhãn giảm giá; hiển thị so sánh giá chỉ khi lịch sử giá hợp lệ.

Quota tổng và per-buyer được reserve **cùng transaction checkout** với stock và voucher, không khi add cart/quote. Reservation `held → committed/released/expired`, unique order item+campaign, atomic conditional update để không oversell; worker expiry phối hợp payment state. Quote gắn campaign version và expiry ngắn, checkout stale trả `PRICE_CHANGED/FLASH_SALE_ENDED/FLASH_SALE_SOLD_OUT`; client phải xác nhận lại, không tự đổi sang giá thường sau submit. Discount stacking F2 cần policy version: mặc định flash sale không stack item-level promotion/voucher trên cùng item; shipping voucher riêng có thể áp dụng. Hủy/return dùng snapshot allocation, quota đã paid không hoàn lại trừ cancel trước mock settlement theo policy.

Public API list campaign active/scheduled, detail and eligible listing; seller create/list own, admin approve/pause; schema `flash_sale_campaigns/items/reservations` ở [Data model](../system/data-model.md). Test concurrent last quota/stock, timezone boundary, seller price change, duplicate settlement, shop suspension, campaign pause, false crossed-out price, cursor and audit. Bật F2 chỉ sau load test và rollback plan theo [Roadmap](../delivery/roadmap.md).
