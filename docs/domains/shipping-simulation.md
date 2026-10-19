---
doc_id: DOMAIN-SHIPPING
title: Shipping simulation
status: accepted
owner: Fulfillment
last_updated: 2026-10-19
version: 4.1.0
source_of_truth: Phí và vòng đời vận chuyển mô phỏng
---

# Vận chuyển mô phỏng

Checkout chọn phương thức `standard_simulated` hoặc `express_simulated` **riêng từng shop** trong quote; checkout gửi lại cùng selection và server tính lại, snapshot method/fee/policy version trên shop order. Phí là policy versioned server-side; quote nêu phí theo shop, ETA **ước lượng giả lập**. Không tích hợp carrier, không phát tracking number/label thật, không chuyển địa chỉ buyer cho người giao hàng. Sau payment success, tạo một shipment cho mỗi shop order; `ready → packed → shipping_simulated → delivered_simulated`. Worker scheduler dùng deadline cấu hình và clock injection để test; admin có thể chuyển hợp lệ với reason/audit, seller F1 chỉ chuyển của shop được ủy quyền. Mapping chi tiết shop-order/shipment actor và nhánh giao thất bại chưa được Product chốt; xem [audit sản phẩm](../delivery/full-product-audit-2026-10-19.md), không tự cho worker/seller skip state.

Timeline có event append-only; transition cần version và idempotency, không được skip trạng thái hoặc chuyển nếu payment chưa paid. “Delivered” chỉ là sự kiện mô phỏng trong app; email và UI nói rõ. Nếu order cancel trước shipping, shipment `cancelled`; sau đó chỉ đi qua return/refund policy. Address snapshot được hạn chế field ở seller và tuyệt đối không xuất cho carrier.

Phí shipping là VND integer, >=0, được khóa trong order snapshot. Đổi policy sau đặt không sửa đơn cũ. API buyer đọc shipment trong order detail; admin/seller mutation qua shop order transition, không expose worker control ra public. `409 INVALID_STATE_TRANSITION` hoặc `404` cross-shop. Test: scheduler retry, wrong payment state, cancel race, version conflict, ETA locale và PII redaction.


## Quy trình buyer–seller–admin

Seller của shop order paid có capability `orders.manage` được đánh dấu `processing`/`packed` theo UI; Admin có thể xử lý ngoại lệ; Worker tự advance `shipping_simulated`/`delivered_simulated` theo clock **nếu policy tự động được bật**. Giả định người chịu trách nhiệm bấm state hay scheduler là [A-02](../delivery/audit-and-open-decisions.md) cần xác minh; để agent triển khai, mặc định seller xử lý processing/packed, worker advance shipping/delivered sau delay cấu hình. Không có giao hàng vật lý dù seller thao tác trạng thái.

| Transition | Precondition | Error |
|---|---|---|
| `ready → packed` | Payment paid, shop order confirmed/processing, version đúng | 409 state/version |
| `packed → shipping_simulated` | Seller/Admin command hoặc scheduler, không đã cancel | 409/cancel conflict |
| `shipping_simulated → delivered_simulated` | Deadline giả lập và no prior return | 409 |
| `ready/packed → cancelled` | Cancel policy, payment refund/reservation compensation | 409 nếu đã shipping |

Thông báo “đang giao” chỉ sau commit transition shipping_simulated. Timeline/ETA là mô phỏng và không chứa carrier/tracking URL thật. Seller không cần address đầy đủ; UI buyer thấy address snapshot của mình. Test delayed job duplicate, seller/Admin/worker race, fee snapshot, wrong-shop transition và notification exactly-once logical.
