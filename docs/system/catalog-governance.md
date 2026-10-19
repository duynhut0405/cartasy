---
doc_id: CATALOG-GOV
title: Cartasy Catalog and Marketplace Governance
status: accepted
owner: Cartasy Product & Operations
last_updated: 2026-10-19
version: 4.1.0
source_of_truth: Product/listing ownership, publish, moderation, shop và quảng cáo
review_cycle: Khi thêm shop/source/capability hoặc đổi policy nội dung
---

# Quản trị catalog và marketplace

## 1. Mô hình

`Product` là nội dung chuẩn; `Variant` là option/SKU; `Listing` thuộc một shop và một variant, mang giá VND, stock/purchase limit và publish state. Một product có nhiều listing/shop. F1 có nhiều seller shop active; platform có thể có shop mẫu nội bộ, không hardcode ID. Seller F1 có thể đề xuất product mới hoặc dùng product có sẵn; không tự sửa canonical content của shop khác. `ExternalOffer` affiliate sau MVP là tài nguyên đối tác **khác Listing**, không đi vào cart/checkout Cartasy; chỉ hiện CTA outbound có disclosure theo [Affiliate](../domains/affiliate.md).

## 2. Nguồn dữ liệu và quyền

F1 dùng catalog demo do Cartasy tạo hoặc có giấy phép rõ, provenance trên product/media. Không scrape hoặc lấy ảnh/mô tả từ marketplace khác để lấp dữ liệu. Import F2 qua CSV/API/feed chỉ từ nguồn được cấp phép, có source ID, external ID, checksum/version, quyền sử dụng, quota và người chịu trách nhiệm. Raw record vào staging; không ghi thẳng public product/listing. Import retry cùng source/version idempotent; lỗi từng dòng và preview trước commit.

## 3. Lifecycle và publish gate

```text
product: draft → pending_review → published ↔ paused → archived
variant: draft → active → disabled
listing: draft → pending_review → active ↔ paused → archived
media: uploaded → quarantined → approved/rejected → archived
shop: draft → active → suspended → closed
```

Publish product cần category/brand policy, variant active, media có quyền/alt text, mô tả hợp lệ và không trùng lặp rõ. Publish listing cần shop active, variant/product published, giá VND hợp lệ, stock config và policy hiển thị mô phỏng. Listing paused/archived không nhận checkout; order snapshot cũ vẫn đọc được. Admin/Seller mutation dùng version/`If-Match`, reason và audit. Emergency disable shop/listing/promotion có hiệu lực ở checkout ngay, không đợi cache.

## 4. Matching và nhiều shop

Khi seller F1 gửi sản phẩm, matching theo GTIN/brand/model/thuộc tính có confidence và review; không merge chỉ vì tên giống. Nếu dùng product canonical có sẵn, listing vẫn là tài nguyên riêng của shop. Merge/split product là tác vụ System Admin có preview, mapping, alias và audit, không sửa order snapshot. Shop không thể chiếm listing, review hoặc media của shop khác. Một shop có thể bán nhiều variant, một variant có thể được nhiều shop bán.

## 5. Giá, stock và khuyến mãi

Shop listing giữ giá niêm yết, currency VND, effective time và version. Stock chính xác do Inventory module sở hữu; UI public có thể làm tròn/diễn đạt, checkout đọc primary. Giảm giá/promotion có owner shop hoặc platform, điều kiện thời gian/số lượt/giới hạn và thứ tự áp dụng xác định trong Pricing module. Không cho giá âm hoặc giá gạch ngang hư cấu. Nếu không có cơ sở giá trước giảm, không dùng “giảm x%”. Quảng cáo F3 không được làm thay đổi giá checkout ngoài promotion công khai.

## 6. Liên kết nghiệp vụ marketplace và quảng cáo

[Marketplace/Seller](../domains/marketplace-seller.md) sở hữu onboarding, membership, capability và shop state. [Advertising](../domains/advertising.md) sở hữu sponsored placement/disclosure F3. [Reviews](../domains/reviews.md) sở hữu điều kiện đánh giá trải nghiệm mô phỏng và moderation; không trình bày review như xác nhận đã nhận/dùng hàng thật. Tài liệu này chỉ sở hữu provenance, quyền nội dung, publish gate và takedown của catalog/media.

Nội dung/ảnh/giá/stock external offer phải có nguồn cấp phép và thời điểm cập nhật; quyền dùng API/feed của đối tác không được suy từ một link affiliate. Nếu partner thu hồi quyền hoặc offer hết hạn, ẩn CTA/asset liên quan; không tái sử dụng ảnh đối tác làm canonical product của Cartasy. Sponsored creative cũng phải có quyền dùng media và link/claim được duyệt.

## 8. Chỉ số vận hành

Coverage product có listing hợp lệ; tỷ lệ media có quyền/alt; số listing paused/out of stock; stock conflict tại checkout; thời gian moderation; duplicate rate; seller violation/takedown; ad disclosure compliance. Dashboard seller là aggregate theo shop, không có hành vi cá nhân của buyer.
