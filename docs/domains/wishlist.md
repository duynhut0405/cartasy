---
doc_id: DOMAIN-WISHLIST
title: Wishlist
status: accepted
owner: Customer Intent
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Lưu sản phẩm yêu thích và đồng bộ
---

# Wishlist

Buyer đăng nhập lưu tối đa 500 product vào một wishlist mặc định; unique `buyer_id+product_id`. Wishlist theo **product**, không theo listing, để không mất mục khi shop đổi giá; lưu `preferred_variant_id/listing_id` tùy chọn chỉ như preference. Guest giữ product ID trong local storage tối đa 100 mục, không lưu dữ liệu cá nhân. Sau login, client gửi batch merge idempotent; server bỏ product không còn public và trả danh sách `added/already_present/unavailable`. Không tự thêm wishlist vào cart.

Thao tác add/remove idempotent: add trùng trả entry hiện có, remove thiếu vẫn thành công; API chỉ lấy buyer từ session. Product archive vẫn có thể hiển thị trong wishlist với nhãn unavailable và nút xóa, nhưng không cho add cart nếu không có listing active. Price/availability hiện tại lấy catalog read model; email price drop F2 phải opt-in và chống spam. Thứ tự newest-first, cursor; mục riêng tư không index/search công khai.

Web có nút tim trên card/detail và trang `/wishlist`; trạng thái optimistic chỉ finalize khi server trả thành công, rollback khi `401/409/503`. API `GET/POST /v1/me/wishlist`, `DELETE /v1/me/wishlist/:productId`, `POST /v1/me/wishlist/merge`; error chuẩn theo [API contract](../system/api-contracts.md). Event `wishlist.item_added/removed` cho analytics chỉ khi consent phù hợp, không dùng làm nguồn dữ liệu. Test: merge lặp, concurrent add/remove, cross-user ID, archived product, guest privacy và limit.


## Contract và đồng bộ

| Hành động | Request/response | Bất biến |
|---|---|---|
| Add/remove | Product ID, optional preferred listing; server trả current entry | Unique buyer+product; thao tác lặp idempotent |
| Guest merge | Versioned product IDs + `Idempotency-Key`; trả added/already/unavailable | Không nhân bản, tối đa 500 server, guest không gửi giá |
| List | Cursor, current eligible listing/price hint | Wishlist private, archive product vẫn hiển thị unavailable |

Buyer có thể theo dõi shop hoặc category trong F2 nếu sản phẩm yêu cầu; MVP **không** tự suy sở thích từ wishlist để gửi campaign marketing. Notification sản phẩm mới chỉ theo opt-in/interest người dùng chủ động chọn ở [Communications](communications.md). Seller/Admin không đọc danh sách wishlist theo người dùng; reporting chỉ aggregate/privacy threshold. Test merge sau login trên hai tab, concurrent remove/add, archived listing, session logout/local cleanup, cross-user access.
