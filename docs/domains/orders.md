---
doc_id: DOMAIN-ORDERS
title: Order và shop order
status: accepted
owner: Orders
last_updated: 2026-10-19
version: 4.1.0
source_of_truth: Order hierarchy, state và hủy đơn
---

# Đơn hàng

Một checkout tạo một buyer order, một shop order cho mỗi shop, nhiều order item. Từng item lưu immutable snapshot tên sản phẩm/variant/shop, quantity, đơn giá và giảm giá phân bổ; shop order lưu phí vận chuyển, còn buyer order lưu địa chỉ snapshot lúc đặt; catalog/profile thay đổi không sửa lịch sử. Tổng buyer order bằng tổng shop orders; currency VND. Order code thân thiện chỉ để hiển thị, ID opaque cho API.

Parent `pending_payment → confirmed → partially_fulfilled → completed_simulated`; terminal `payment_failed/cancelled`. Shop order `pending_payment → confirmed → processing → packed → shipping_simulated → delivered_simulated`; nhánh `cancelled/returned_simulated` theo policy. Parent được tính từ child và payment trong transaction, không cho admin sửa text trực tiếp. Lịch sử state append-only chứa actor, time, reason code, idempotency key. UI luôn gắn nhãn trạng thái vận chuyển/thanh toán mô phỏng.

Buyer đọc/hủy order của mình. Hủy toàn đơn chỉ nếu mọi shop order đủ điều kiện; hủy một shop order sau paid và trước shipping_simulated tạo refund mô phỏng theo snapshot allocation, restock theo policy. Sau shipping_simulated dùng return request, không hủy trực tiếp. Admin vận hành có capability và audit reason; seller F1 chỉ xem/chuyển shop order của shop mình, không sửa payment hoặc parent. Worker chuyển fulfillment theo policy/scheduler, không vượt trạng thái khi payment pending.

`GET /v1/orders` cursor, `GET /v1/orders/:id` và `POST .../cancel` theo [API contracts](../system/api-contracts.md). Khác owner trả `404`, version/state sai `409`, thiếu quyền `403`. Web hiện timeline, item snapshot, **xác nhận đơn mô phỏng** (không gọi hóa đơn), ticket/return actions và notification. Admin filter theo state/shop/date, tránh lộ PII ngoài quyền. Test: multi-shop aggregate, immutable snapshot, concurrent cancel/ship, tenant read, idempotency và status history.


## Permission và transition matrix

| Actor | Xem | Command |
|---|---|---|
| Buyer | Parent order và mọi shop order của chính mình | Hủy toàn/từng shop order trong cửa sổ hợp lệ; return sau shipping |
| Seller | **Chỉ** shop order của shop có `orders.manage`; buyer PII redacted | `confirmed → processing → packed → shipping_simulated` theo policy/`If-Match` |
| Operations Admin | Order/shop order theo capability | Duyệt ngoại lệ/hủy/transition có reason/audit |
| Worker | Aggregate theo event/job | Advance shipment theo clock/policy, idempotent |

Parent order là aggregate kết quả từ payment + shop-order states; buyer không thể sửa parent text. Seller transition không được nhảy qua payment pending hoặc shop suspended policy; shop suspend sau order tạo không tự hủy đơn đã paid, Admin quyết định handling. API list theo buyer/shop cursor, status filter enum, stable sort. `404` khi ID khác owner/shop, `409` version/transition race, `403` thiếu capability. Timeline event append-only và notification sau commit; test hai seller cùng đơn, cross-shop export, duplicate transitions và refund allocation.
