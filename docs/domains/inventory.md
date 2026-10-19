---
doc_id: DOMAIN-INVENTORY
title: Inventory và reservation
status: accepted
owner: Inventory
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Tồn kho, giữ chỗ, nhập/xuất và đối soát
---

# Inventory

Tồn kho thuộc listing/shop/location logic; F1 một location `virtual_fulfillment`, F2 có thể thêm location mà không đổi order item. `on_hand >= reserved >= 0`, `available = on_hand - reserved`. Public search chỉ hiện availability hint; checkout đọc stock authoritative trên primary. Admin/seller adjustment cần delta, reason code, actor, shop capability và version; không sửa số stock trực tiếp. Inventory movement append-only; điều chỉnh âm không được làm `on_hand < reserved`.

Checkout reserve từng listing trong cùng transaction tạo order, khóa stock rows theo listing ID tăng dần và dùng conditional update/CAS. Reservation `held → committed` khi payment success hoặc `held → released/expired` khi failure/cancel/TTL. Mỗi reservation chỉ kết thúc một lần; commit giảm cả on_hand và reserved, release chỉ giảm reserved. Đơn trả hàng được restock theo quyết định return, không tự động khi request mới tạo; restock idempotent theo return item. Listing archive không xóa stock history.

Worker quét reservation hết hạn theo batch `SKIP LOCKED`, nhưng phải phối hợp payment timeout state: không release nếu payment vừa paid; transaction khóa/cas. Audit/reconciliation định kỳ đối chiếu stock, reservations, movements và order effects; sai lệch tạo alert, không tự âm thầm sửa. Lỗi `OUT_OF_STOCK`, `STOCK_VERSION_CONFLICT`/`CONFLICT`, `INVALID_ADJUSTMENT`/`422`. Admin stock UI hiển thị on_hand/reserved/available, movement và cảnh báo low stock; seller F1 chỉ shop mình. Test concurrency last unit, duplicate settlement/return, negative adjustment, expiry race và reconciliation.


## Adjustment, reservation và quyền

| Operation | Điều kiện | Tác động atomic |
|---|---|---|
| Seller adjustment | Membership active + `inventory.manage`, listing cùng shop, reason/`If-Match` | movement append-only, `on_hand + delta >= reserved` |
| Checkout hold | Listing/shop eligible và `available >= qty` | reserved += qty, reservation held unique order item |
| Mock success | Held, payment pending | on_hand -= qty, reserved -= qty, reservation committed |
| Failure/expiry | Held, payment failed/cancelled | reserved -= qty, reservation released/expired |
| Return restock | Return approved + explicit restock policy | on_hand += qty một lần theo return item reference |

Stock của từng shop độc lập dù cùng variant; seller A không xem/điều chỉnh stock B. Location ảo một/listing là giả định kỹ thuật, thêm multi-location cần mapping/fulfillment decision chứ không sửa stock số tự do. Worker reconciliation so movement/reservation/order effects và alert; không auto-heal. API inventory scoped ở [API contract](../system/api-contracts.md), schema/index ở [Data model](../system/data-model.md).
