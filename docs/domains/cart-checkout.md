---
doc_id: DOMAIN-CART-CHECKOUT
title: Cart, quote và checkout
status: accepted
owner: Cart & Checkout
last_updated: 2026-10-19
version: 4.1.0
source_of_truth: Giỏ hàng, định giá lại và tạo đơn nguyên tử
---

# Cart và checkout

## Cart

Cart active của buyer là server-owned; một buyer tối đa một cart active. Guest cart chỉ lưu `listingId+quantity` ở browser, giới hạn 100 dòng và TTL local 30 ngày; sau đăng nhập merge một lần bằng key, giới hạn qty theo policy. Cart item tham chiếu listing (không tham chiếu product chung). Quantity 1–99 và không vượt available stock khi quote/commit; add có thể giữ item tạm thời dù stock vừa hết, UI gắn nhãn unavailable. Giá hiển thị trong cart là dữ liệu đọc lại, không đóng băng. Update/remove dùng cart version; client hiển thị conflict và refresh. Coupon code trên cart là ý định áp dụng, chưa chiếm lượt.

## Quote

Quote lấy cart version, address ID, **bảng chọn shipping method theo từng shop có item trong cart**, coupon code, payment method giả lập. Mỗi shop đúng một lựa chọn `standard_simulated` hoặc `express_simulated` nếu method đó hợp lệ với shop/địa chỉ; thiếu/thừa shop ID hoặc method không hợp lệ trả `422`, shop/listing bị khóa trả conflict. Server tính theo thứ tự: giá listing × quantity → item/shop subtotal → discount đủ điều kiện → phí vận chuyển mô phỏng → total VND; phân bổ discount bằng integer, phần dư theo stable order item ID. Quote không giữ stock/voucher, TTL 5 phút, gồm fingerprint của cart, listing prices/versions, promotion policy, **shipping selection/policy từng shop** và expected total. Giá về client là string decimal minor VND. Quote có `simulationNotice` rõ ràng.

## Place order

`POST /v1/checkouts` yêu cầu session, cart version, quote fingerprint, expected total, address thuộc buyer, **cùng shipping selection per shop đã quote**, phương thức mock và `acknowledgedSimulation: true`, cộng `Idempotency-Key`. Server lock cart và inventory theo thứ tự ID, đọc giá/promotion/shipping policy authoritative, validate lại; nếu selection/fee thay đổi trả quote mới để buyer xác nhận, không âm thầm đổi method. Trong **một transaction**: đóng cart, reserve stock, hold voucher, tạo order/shop orders/items/address, shipping method/fee policy snapshot per shop, payment pending, idempotency record và outbox. Nếu lỗi rollback toàn bộ. Same key+same payload trả cùng order; key+payload khác `409`. Key khác trên cart đã đóng không tạo đơn thứ hai. Timeout mạng: client giữ key, tra `GET /v1/checkouts/by-key/:key`, không tạo checkout mới.

## Hành vi lỗi

`PRICE_CHANGED` trả quote mới để xác nhận lại; `OUT_OF_STOCK` nêu item cần sửa; `COUPON_INELIGIBLE` cho bỏ mã; `CART_CHANGED` refresh; `UNAUTHENTICATED` giữ guest cart và đưa login; `PAYMENT_PENDING` hiển thị trạng thái và poll order. Không retry mutation bằng key mới nếu chưa xác định kết quả. Web step: giỏ → địa chỉ/phương thức/quote → xác nhận mô phỏng → order pending/result. API không tin `userId, price, shippingFee, discount` từ client. Buyer không thể checkout listing của shop bị khóa.

Giao tiếp owner: Catalog/Listing đọc giá, Inventory reserve, Promotions hold, Orders tạo snapshot, Payments tạo pending, Outbox ghi event; [business flows](../system/business-flows.md) và [data model](../system/data-model.md) giữ invariant xuyên module. Test: concurrency last-stock, duplicate keys, price/coupon change, rollback, mixed-shop totals, network timeout và guest merge.


## Use case matrix và quyền

| Use case | Actor | State precondition | Kết quả |
|---|---|---|---|
| Guest add/update | Guest browser | Listing public hint | Local ID/qty only; login merge |
| Server cart mutation | Buyer | Cart active, listing eligible, version đúng | Cart version tăng; server trả giá hint mới |
| Quote | Buyer | Address owner, cart active, shipping method valid per shop | Breakdown + fingerprint/expiry, không reserve |
| Place order | Buyer | Ack mô phỏng, quote/cart hiện hành, shop active, đủ stock/quota | One order + N shop orders/payment pending/outbox trong transaction |

Mỗi shop phải có shipping method/fee riêng; buyer không chọn một phương thức giả làm thay mọi shop nếu policy khác. Hủy request HTTP giữa chừng không có nghĩa transaction rollback: tra by-key. Không gửi order confirmation khi checkout chưa commit. Catalog/Inventory/Promotion/Orders/Payments cung cấp port transaction-aware; Checkout chỉ orchestration và idempotency. Test hai seller cùng product variant, một shop suspend giữa quote và commit, coupon quota last unit, duplicate cross-device click, transaction rollback và network timeout sau commit.
