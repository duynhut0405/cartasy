---
doc_id: DOMAIN-CMS
title: CMS and Merchandising Content
status: accepted
owner: Content Operations
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Nội dung home/banner/page, lịch xuất bản và an toàn hiển thị
---

# CMS và nội dung marketplace (MVP)

CMS sở hữu banner/home modules, landing page và trang tĩnh liên quan sản phẩm. Không thay catalog/product/price bằng text CMS. `cms_pages` và `cms_blocks` có `draft → pending_review → scheduled → published → archived`; publish cần content admin + approver khác nếu block dẫn tới promotion/claim quan trọng. Version và lịch sử bất biến; rollback tạo revision mới, không xóa lịch sử. Public chỉ đọc revision published trong hiệu lực; cache theo tag và event invalidation.

Block type allowlist: hero, category grid, product collection, shop collection, editorial text, announcement. Product/shop/listing reference phải active/published ở lúc render; block tự ẩn/fallback khi reference bị suspend/unpublish. Campaign sản phẩm mới có thể tham chiếu CMS landing đã publish, nhưng CMS publish không tự gửi noti. Banner image phải từ media approved, có alt text, crop responsive, provenance/license; URL nội bộ allowlist, external URL qua review và rel=noreferrer. Rich text giới hạn schema, sanitize ở server, không lưu/serve script/iframe/HTML tự do.

Admin route tạo/sửa/preview/review/schedule/publish/rollback với `If-Match`, audit actor/reason. Seller không sửa home platform; có thể cập nhật mô tả shop do `shops` sở hữu, hoặc đề xuất collection shop nếu policy cho phép F2. F1 có `vi`/`en` cho public Web: mỗi page có translation group và revision/slug/block text/media alt theo locale; home modules publish độc lập theo locale. Bản chưa dịch/chưa duyệt không public ở locale đó, không fallback âm thầm. Biên tập viên preview/diff và publish từng locale; link nội bộ trỏ bản dịch published hoặc fallback route đã duyệt. UTC trong DB, preview timezone người biên tập. Không dùng CMS để dựng giá gạch ngang/đánh giá/lượt bán giả. API public `GET /v1/home?locale=...`, `GET /v1/pages/:slug?locale=...`; admin `/v1/admin/cms/*` ở [API contract](../system/api-contracts.md). Lỗi `404` unpublished, `409` version/state/reference invalid, `422` schema/link/media. Test preview vs public, cache invalidation, expired schedule, XSS, seller suspension, rollback, SEO canonical/hreflang/noindex và accessibility.


## Contract triển khai

| Operation | Quyền | Input/điều kiện | Kết quả |
|---|---|---|---|
| Tạo/sửa draft | `cms.write` + MFA | `slug` unique theo locale, block allowlist, `If-Match` khi sửa | Revision mới; public không đổi |
| Submit/review | `cms.review`; reviewer khác author nếu claim quan trọng | Không còn media/reference lỗi, preview được ghi nhận | `pending_review → scheduled/published` hoặc rejected có reason |
| Publish/schedule | `cms.publish` | `startsAt < endsAt` nếu có, revision đúng bản duyệt | Public revision chuyển nguyên tử; event invalidate cache |
| Rollback/archive | `cms.publish` + reason | Revision đích hợp lệ; giữ audit | Tạo revision mới hoặc gỡ public; không xóa lịch sử |

Ví dụ page home gồm các block versioned theo vị trí: `hero` với image/alt/link, `category_grid` với category IDs, `product_collection` với ID allowlist và `editorial_text`. JSON block có schema/version và maximum payload, số block/page và chiều dài text được chốt trong DTO có test; tránh lưu HTML tùy ý. Product/listing/shop reference được recheck tại render hoặc projection; block không hợp lệ ẩn riêng block và ghi metric, không làm hỏng cả home. Public response có `revisionId, publishedAt, locale` và cache key; staging preview dùng signed admin-only URL và `noindex`, không share draft qua public API.

Endpoint chuẩn: `GET /v1/home`, `GET /v1/pages/:slug`; `GET/POST /v1/admin/cms/pages`, `GET/PATCH /v1/admin/cms/pages/:id`, `POST /v1/admin/cms/pages/:id/submit`, `POST /v1/admin/cms/pages/:id/decision`, `POST /v1/admin/cms/pages/:id/publish`, `POST /v1/admin/cms/pages/:id/rollback`. `409 VERSION_CONFLICT/INVALID_STATE_TRANSITION` cho revision cũ hoặc transition sai, `422 VALIDATION_FAILED` cho block/media/link, `404` cho unpublished/cross-scope. `cms_pages/blocks/revisions` là source, `cms.published/archived` outbox điều khiển invalidation; scheduler publish idempotent theo revision. Admin UI có preview desktop/mobile, diff, thời gian theo timezone, audit và confirmation cho publish/rollback.
