---
doc_id: DOMAIN-CATALOG
title: Product, category, variant, listing và media
status: accepted
owner: Catalog & Listings
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Quy tắc catalog và listing
---

# Catalog và listing

## Mô hình và trạng thái

`Product` là identity/nội dung chuẩn (brand, category, attributes, media; name/slug theo locale); `Variant` là SKU/option tổ hợp; `Listing` là lời chào bán của một shop cho một variant, sở hữu giá VND và trạng thái bán. Từ F1 đã có `shop_id`. Product `draft → pending_review → published ↔ paused → archived`; listing `draft → pending_review → active ↔ paused → archived`. F1 admin có thể duyệt platform product/listing trong cùng phiên với reason/audit; F1 seller gửi duyệt, không tự active. Chỉ product published + category active + variant active + shop active + listing active mới public. Unpublish product làm listing biến mất khỏi search nhưng không sửa order snapshot.

Category là cây có `parent_id`, depth tối đa 4 và không cycle; slug unique trong parent **và locale**; di chuyển category cần kiểm tra ảnh hưởng URL/filter. Variant option keys phải theo schema category, fingerprint unique trong product. Giá listing là integer minor VND, `>= 0`; không có “giá product chung”. Một shop có tối đa một listing active cho variant; thay đổi giá dùng version, audit actor/reason, phát `listing.price_changed`. Buyer luôn thấy shop và giá của listing được chọn.

F1 có `vi`/`en`: product/category/shop là identity theo ID ổn định, còn title/description/SEO/slug/alt theo locale và được duyệt riêng. Unique slug product/shop là trong locale. Translation có draft/review/published, version và audit; public cần cả resource gốc lẫn translation cùng locale published. `vi` được backfill từ nội dung hiện có; không tự xuất bản `en` bằng fallback hoặc dịch máy. Slug cũ redirect trong cùng locale; đổi slug/category cập nhật canonical, hreflang, sitemap, search và cache. Giá VND/variant/listing/stock không dịch và không thay đổi theo locale. [Data model](../system/data-model.md), [ADR-0012](../decisions/ADR-0012-multilingual-storefront.md).

Trước release F1, home/điều hướng cốt lõi và các trang pháp lý public phải có bản `vi`/`en` đã duyệt; catalog item chỉ xuất hiện trong collection của locale khi translation item đó published. Order snapshot giữ tên gốc lúc đặt, không bị dịch lại sau khi catalog đổi.

## Validation và quản trị

Product name từng locale 3–200 ký tự, description từng locale sanitize và giới hạn 20.000 ký tự, slug canonical theo locale, tối đa 20 media đã scan; ảnh có alt text theo locale, thứ tự, ownership/license. Không nhận HTML/script tùy ý; media upload private qua signed URL ngắn hạn, MIME/checksum/size kiểm tra ở server và worker, duyệt trước publish. Category/attribute schema điều khiển filter; thay đổi schema có migration/backfill. Admin duyệt canonical product, category, brand, media; seller F1 đề xuất product mới hoặc gắn listing vào product đã có. Seller chỉ sửa listing của shop mình; admin có quyền moderation và phải ghi lý do.

## Đọc công khai và lỗi

Trang home/category/product/shop chỉ trả nội dung public; slug cũ có redirect đã duyệt, slug trùng trả `409`. Listing giá/stock có thể đổi sau khi load; add cart kiểm tra lại. Lỗi `CATALOG_NOT_PUBLISHED`, `LISTING_UNAVAILABLE`, `INVALID_VARIANT_OPTIONS`, `MEDIA_NOT_READY`, `VERSION_CONFLICT` map 409/422 theo [API contract](../system/api-contracts.md). Admin form giữ draft và chỉ publish khi đủ category, variant, media, listing/stock policy. Sự kiện `catalog.product_published`, `catalog.product_unpublished`, `listing.changed` làm mới search/cache projection; order không phụ thuộc vào cache. Chuẩn nội dung/moderation tại [Catalog governance](../system/catalog-governance.md).

Test: product/listing state cross-check, slug uniqueness/redirect theo locale, thiếu bản dịch 404, variant fingerprint, tenant ownership, XSS/media quarantine, archive có order lịch sử, published page SEO/hreflang.


## Contract triển khai MVP

| Action | Actor | Điều kiện/validation | Side effect |
|---|---|---|---|
| Tạo canonical product/variant | Admin hoặc seller proposal | Category/brand/schema/slug/option fingerprint, provenance/media quyền sử dụng | Draft hoặc pending_review, audit |
| Tạo listing/giá | Seller của shop active + `listing.manage` | Product/variant published, VND integer >=0, unique shop+variant active, version | Listing draft/pending_review, price history |
| Duyệt/publish/pause | Admin moderation | Media approved, shop active, content policy, reason, `If-Match` | Search/cache invalidate, notification seller |
| Đọc public | Guest/buyer | Product/category/variant/listing/shop eligible | Chỉ media approved; giá hiển thị gắn listing/shop |

Seller MVP có thể gửi product proposal, không sửa nội dung canonical của shop khác; Admin quyết định merge/match dựa GTIN/brand/model, không chỉ tên. Shop suspend làm listing public biến mất ngay ở API/checkout, projection cập nhật eventual nhưng guard đọc chặn. Archive không xóa order snapshot hoặc redirect slug cũ. API owner ở [API contract](../system/api-contracts.md), schema ở [Data model](../system/data-model.md). Test hai shop cùng variant/giá khác nhau, publish race, price change stale quote, media XSS/provenance và slug redirect.

`ProductProposal` F1 là aggregate riêng thuộc Catalog: seller shop active tạo/sửa draft với title/category/brand/variant/media/provenance; `draft → pending_review → approved/rejected/needs_changes`. Admin `catalog.review` có thể match tới canonical product/variant đã có hoặc tạo canonical draft qua owner use case; quyết định có reason, reviewer, version và audit. `needs_changes` trả cho seller sửa rồi submit lại; approval không tự publish listing hoặc thay product của shop khác. Unique/idempotency business key ngăn duplicate decision; media chỉ đi tiếp khi quyền/scan đạt. `product_proposals` ở [Data model](../system/data-model.md), seller/admin routes ở [API contract § 12](../system/api-contracts.md). Test match sai SKU, hai proposal cùng product, reviewer conflict, media thiếu quyền và seller cross-shop.
