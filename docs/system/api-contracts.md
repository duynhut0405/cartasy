---
doc_id: API-CONTRACT
title: Cartasy API Contracts
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: HTTP endpoint, payload, error, auth và concurrency contract
review_cycle: Mỗi thay đổi API
---

# Hợp đồng API v1

## 1. Quy ước

Base path `/v1`, JSON UTF-8, field `camelCase`, thời gian ISO 8601 UTC. ID opaque UUID string; tiền `{ amountMinor: "125000", currency: "VND" }`, không dùng JSON number cho tiền. Product/listing public có SEO slug nhưng mutation và quan hệ dùng ID. Response business thành công trả object/resource rõ; lỗi dùng envelope chuẩn. Auth route của Better Auth giữ payload provider và được contract-test riêng.

Header: `X-Request-Id` trên mọi response; `Idempotency-Key` (UUID/ULID opaque) cho checkout/operation retry; `If-Match` ETag/version cho mutation có nguy cơ ghi đè. Cookie session `HttpOnly` host-only ở API; browser Web/Admin chung Seller-Platform gọi typed client với `credentials: include` và Origin được allowlist. Trang private không dựa vào SSR session trên web/admin origin. Không nhận `userId`/`shopId` tự do thay cho actor/resource policy.

List dùng `limit` 1–100, `cursor` opaque gắn filter/sort; response `{ data: [], page: { nextCursor: null } }`. Cursor sai/đổi filter trả `422 VALIDATION_FAILED`. API không expose offset sâu. Route public chỉ trả published/active content.

### Locale của public read

Các GET public `/home`, `/pages/:slug`, `/categories*`, `/products*`, `/shops*` nhận query `locale=vi|en` **bắt buộc**. Web lấy giá trị này từ route `/[locale]`; API không đoán từ browser header. `locale` ngoài allowlist trả `422 VALIDATION_FAILED`; bản dịch/resource chưa published ở locale yêu cầu trả `404 NOT_FOUND` cho detail, collection chỉ gồm item có bản dịch published. Response có `Content-Language: vi|en`, `locale`, resource ID ổn định, localized `slug/title/description/alt` và `availableLocales`/localized slug map **chỉ cho bản đã published** để xây locale switcher/hreflang. Cache key/search index/cursor bao gồm locale; publish/unpublish/đổi slug invalidate đúng locale và link liên quan. Giá/stock/listing ID, VND và auth policy không đổi theo locale. API trả `Cache-Control` thích hợp cho public; private luôn `no-store`. OpenAPI khai báo enum, response và lỗi; không dịch error `code`, chỉ UI dịch message theo code.

## 2. Error envelope

```json
{
  "error": {
    "code": "PRICE_CHANGED",
    "message": "Giá đã thay đổi. Vui lòng xem lại đơn.",
    "details": { "quote": "..." },
    "requestId": "..."
  }
}
```

| Code | HTTP | Ý nghĩa |
|---|---:|---|
| `VALIDATION_FAILED` | 422 | Input không đúng schema |
| `UNAUTHENTICATED` | 401 | Thiếu/hết session |
| `FORBIDDEN` | 403 | Không đủ quyền hoặc shop capability |
| `NOT_FOUND` | 404 | Không có hoặc không thuộc actor; tránh lộ tài nguyên |
| `CONFLICT` | 409 | Idempotency payload khác hoặc xung đột chung |
| `VERSION_CONFLICT` | 409 | `If-Match`/resource version cũ |
| `STOCK_VERSION_CONFLICT` | 409 | Version stock cũ khi điều chỉnh |
| `CATALOG_NOT_PUBLISHED` | 409 | Product chưa đủ điều kiện public |
| `INVALID_VARIANT_OPTIONS` | 422 | Tổ hợp option không hợp lệ |
| `MEDIA_NOT_READY` | 409 | Media chưa qua scan/duyệt |
| `INVALID_ADJUSTMENT` | 422 | Điều chỉnh tồn kho sai hoặc làm âm available |
| `PRICE_CHANGED` | 409 | Giá/discount/phí không còn khớp quote đã xác nhận |
| `LISTING_UNAVAILABLE` | 409 | Listing/variant/shop không public/hợp lệ |
| `OUT_OF_STOCK` | 409 | Tồn kho khả dụng không đủ |
| `CART_CHANGED` | 409 | Cart version không còn khớp |
| `COUPON_INELIGIBLE` | 409 | Mã không áp dụng/hết lượt |
| `PAYMENT_PENDING` | 202/409 | Kết quả mock payment chưa xác định, client poll order |
| `INVALID_STATE_TRANSITION` | 409 | Hành động không hợp lệ với trạng thái hiện tại |
| `RATE_LIMITED` | 429 | Vượt ngưỡng |
| `SERVICE_NOT_READY` | 503 | Dependency bắt buộc chưa sẵn sàng |
| `REVIEW_NOT_ELIGIBLE` | 409 | Đơn/item chưa đủ điều kiện đánh giá mô phỏng |
| `RETURN_WINDOW_CLOSED` | 409 | Quá thời hạn tạo return |
| `RETURN_QUANTITY_EXCEEDED` | 409 | Số lượng đã có request/hoàn vượt item |
| `SHOP_SUSPENDED` | 423 | Shop bị khóa; mutation seller không khả dụng |
| `AUDIENCE_INVALID` | 422 | Target/consent của campaign không hợp lệ |
| `CAMPAIGN_NOT_APPROVED` | 409 | Chưa duyệt hoặc version nội dung đã đổi |
| `PUSH_SUBSCRIPTION_INVALID` | 422 | Endpoint/key/permission không hợp lệ |
| `REPORT_DUPLICATE` | 409 | Report cùng actor/resource/reason đang mở |
| `MODERATION_CONFLICT` | 409 | Quyết định tự duyệt hoặc action/version xung đột |
| `FLASH_SALE_SOLD_OUT` | 409 | Hết quota F2 |
| `FLASH_SALE_ENDED` | 409 | Sale kết thúc F2 |
| `INTERNAL_ERROR` | 500 | Lỗi không mong đợi, không lộ chi tiết |

Error `message` dành cho người dùng, client điều hướng theo `code`. Không phản chiếu SQL/provider error, URL có token hoặc địa chỉ vào response/log.

## 3. Auth và tài khoản

Better Auth mount ở `/v1/auth/*` cho signup, verification, sign-in, sign-out, session, recovery và MFA admin/seller. Adapter/version được pin và snapshot route/payload; không mặc định mọi route auth theo envelope Cartasy. `GET /v1/me` lấy actor từ session; update chỉ cho field allowlist. F1 email/password và seller MFA; OAuth là capability sau khi có provider/test và account-link policy.

| Method | Path | Auth | Hành vi |
|---|---|---|---|
| GET | `/me` | Buyer | Profile allowlist |
| GET | `/portal/context` | Session + MFA theo action | Platform capabilities và active shop memberships tối thiểu cho Admin workspace navigation; không là bằng chứng quyền cho mutation, `no-store` |
| PATCH | `/me` | Buyer | Sửa tên/locale; `If-Match` |
| GET | `/me/addresses` | Buyer | Địa chỉ của mình |
| POST | `/me/addresses` | Buyer | Tạo địa chỉ mô phỏng |
| PATCH/DELETE | `/me/addresses/:id` | Owner | Sửa/ẩn địa chỉ, không sửa order snapshot |
| POST | `/me/data-exports` | Owner + recent auth | Tạo yêu cầu xuất dữ liệu cá nhân; idempotent |
| GET | `/me/data-exports/:id` | Owner | Trạng thái và private expiring URL khi hoàn tất |
| POST | `/me/deletion-requests` | Owner + recent auth | Yêu cầu xóa/ẩn danh theo retention/hold, không hard-delete order |
| GET | `/me/deletion-requests/:id` | Owner | Trạng thái/hold reason an toàn |

Notification và preference nằm ở mục 8.

Địa chỉ là dữ liệu cá nhân dù không giao hàng; UI giải thích không dùng để ship. Response address không gửi cho shop F1 vì không có fulfillment thật.

## 4. Catalog và search

| Method | Path | Auth | Hành vi |
|---|---|---|---|
| GET | `/home` | Public | Module CMS trang chủ đã publish |
| GET | `/pages/:slug` | Public | Trang CMS published theo locale |
| GET | `/categories`, `/categories/:slug` | Public | Category tree và landing |
| GET | `/products` | Public | `q/category/brand/shop/minPrice/maxPrice/sort/cursor` |
| GET | `/products/:slug` | Public | Product, variant, media đã duyệt |
| GET | `/products/:slug/listings` | Public | Listing active theo variant/shop, giá/tồn kho/currency |
| GET | `/shops/:slug` | Public | Profile shop active và listing |
| GET | `/shops/:slug/products` | Public | Product/listing của shop |

Mọi path trong bảng này là dưới `/v1` và dùng `?locale=vi|en`; `:slug` được resolve trong locale tương ứng. Slug cũ chỉ redirect theo mapping đã duyệt, không đoán bằng text của locale khác. Search query/filter/sort/cursor duy trì trong locale, không index nội dung thiếu translation.

Product detail không có giá “chung” vô chủ: `displayPrice` là min price của listing hợp lệ và có `from`, detail luôn cho biết shop/listing được chọn. Public stock là `available/in_low_stock/unavailable` hoặc số lượng nếu policy cho phép; checkout dùng stock chính xác trên primary. Search F1 PostgreSQL full-text/trigram; query và filter được allowlist.

## 5. Giỏ và quote

| Method | Path | Auth | Hành vi |
|---|---|---|---|
| GET | `/cart` | Buyer | Active cart và item đã đọc lại |
| POST | `/cart/items` | Buyer | Add listing/quantity; idempotency |
| PATCH | `/cart/items/:id` | Owner | Set quantity với `If-Match` cart version |
| DELETE | `/cart/items/:id` | Owner | Remove |
| POST | `/cart/merge` | Buyer | Merge guest cart versioned; idempotency |
| POST | `/checkout/quotes` | Buyer | Quote theo cart/address/coupon và `shippingSelections[]` đúng từng shop, không reserve |

Guest cart không có API lưu. Merge body `{ version: 1, items: [{ listingId, quantity }] }`; response trả cart, `merged/adjusted/removed` cùng reason. Không gửi client price hoặc user ID. Quote request F1 có `{ cartId, cartVersion, addressId, couponCode?, paymentMethod, shippingSelections: [{ shopId, method }] }`; `method` là `standard_simulated|express_simulated`. Danh sách phải chứa đúng một entry cho mỗi shop trong cart, không nhận shop khác; server xác minh shop/method/địa chỉ và tính phí. Quote gồm `cartVersion`, `quoteFingerprint`, `expiresAt`, breakdown từng shop (kèm `shippingMethod`, `shippingPolicyVersion`, `shippingFee`) và tổng; không đảm bảo giá/tồn kho tới lúc commit. Checkout gửi lại cùng `shippingSelections` và fingerprint, server tính lại; đổi fee/method/eligibility trả `PRICE_CHANGED` hoặc `LISTING_UNAVAILABLE` kèm quote mới khi an toàn.

Ví dụ quote:

```json
{
  "cartVersion": 7,
  "quoteFingerprint": "opaque-hash",
  "currency": "VND",
  "shops": [
    {
      "shopId": "uuid",
      "subtotal": "250000",
      "discount": "20000",
      "shippingMethod": "standard_simulated",
      "shippingPolicyVersion": 2,
      "shippingFee": "15000",
      "total": "245000"
    }
  ],
  "total": "245000",
  "simulationNotice": "Không thu tiền và không giao hàng thật."
}
```

## 6. Checkout, payment và order

| Method | Path | Auth | Hành vi |
|---|---|---|---|
| POST | `/checkouts` | Buyer | Tạo order/payment pending từ cart; `Idempotency-Key` |
| GET | `/checkouts/by-key/:key` | Buyer | Tra order của chính actor sau timeout |
| GET | `/orders` | Buyer | Lịch sử order của actor |
| GET | `/orders/:id` | Owner | Order, shop order, payment status và timeline |
| POST | `/orders/:id/cancel` | Owner | Hủy khi policy cho phép; idempotency |
| POST | `/orders/:id/shop-orders/:shopOrderId/cancel` | Owner | Hủy phần của một shop; idempotency và refund mô phỏng theo snapshot |
| GET | `/orders/:id/payment` | Owner | Kết quả mock payment/pending |
| GET | `/orders/:id/events` | Owner | Timeline cursor nếu dài |

Checkout request:

```json
{
  "cartId": "uuid",
  "cartVersion": 7,
  "quoteFingerprint": "opaque-hash",
  "expectedTotal": "245000",
  "addressId": "uuid",
  "couponCode": "WELCOME",
  "shippingSelections": [{ "shopId": "uuid", "method": "standard_simulated" }],
  "paymentMethod": "mock_e_wallet",
  "acknowledgedSimulation": true
}
```

Server lấy buyer từ session, validate địa chỉ thuộc buyer, tính lại tất cả, reserve stock và tạo order. `acknowledgedSimulation` là xác nhận UI, không thay security. Success ban đầu trả `201` với `{ orderId, orderStatus: "pending_payment", paymentStatus: "pending" }`. Poll `GET /orders/:id` hoặc dùng push notification khi mock settlement hoàn tất. Same idempotency key + same fingerprint trả cùng order ID/status hiện tại; same key + body khác trả `409 CONFLICT`. Timeout client giữ key và GET order theo response/lookup an toàn, không tạo key mới. Cung cấp `GET /checkouts/by-key/:key` cho actor để tra kết quả sau timeout, chỉ trong retention idempotency.

Payment scenario success/failure/timeout là cấu hình của mock adapter, không nhận từ browser trên staging/production. Test/local có test harness bảo vệ riêng để ép scenario; không expose endpoint điều khiển kết quả ở public. Không có route nhận card number/OTP hoặc callback provider thật.

Payment failure là trạng thái kết thúc của order đó: reservation/coupon hold được release; UI có thể thêm lại item còn hợp lệ vào giỏ mới và tạo checkout mới. Payment timeout giữ order `pending` để đối soát; scheduler F1 kết thúc thất bại sau 15 phút nếu vẫn pending. Không tạo key/order mới khi chưa có kết luận. Hủy toàn đơn chỉ thành công nếu mọi shop order còn đủ điều kiện; hủy từng shop order tạo `payment_refund` theo phân bổ snapshot, trả payment `partially_refunded_simulated` hoặc `refunded_simulated`, không phát sinh chuyển tiền.

Order response snapshot có `simulation: true`, `currency: VND`, totals dạng string, shop order, item, address đã hạn chế field, payment `paid_simulated/pending/failed/partially_refunded_simulated/refunded_simulated` và shipment `shipping_simulated/delivered_simulated`. Các state dùng mã ổn định; UI hiển thị nhãn tiếng Việt có chữ “mô phỏng”.

## 7. Wishlist, review, return và support

| Method | Path | Auth | Hành vi/điều kiện |
|---|---|---|---|
| GET/POST | `/me/wishlist` | Buyer | List cursor/add product public; add trùng idempotent |
| DELETE | `/me/wishlist/:productId` | Owner | Bỏ mục; thiếu mục vẫn idempotent |
| POST | `/me/wishlist/merge` | Buyer | Merge guest IDs với `Idempotency-Key`, trả added/unavailable |
| GET | `/products/:slug/reviews` | Public | Chỉ review published, nhãn simulation và cursor |
| POST | `/orders/:id/items/:itemId/review` | Buyer | Một review/item sau delivered_simulated; `Idempotency-Key` |
| PATCH | `/me/reviews/:id` | Owner | Sửa trong 7 ngày; `If-Match`, về pending_moderation |
| GET | `/me/reviews` | Buyer | List review của actor |
| POST | `/reports` | Authenticated | Báo cáo product/listing/shop/review theo allowlist |
| GET | `/me/reports` | Reporter | Metadata case của mình, không lộ evidence/reviewer |
| GET/POST | `/me/returns` | Buyer | List/tạo request, item quantity và reason; `Idempotency-Key` |
| GET | `/me/returns/:id` | Owner | Detail, decision, refund mô phỏng |
| GET/POST | `/me/support/tickets` | Buyer | List/tạo ticket; rate limit và attachment scan |
| GET | `/me/support/tickets/:id` | Owner | Detail/messages/attachment signed URL có quyền |
| POST | `/me/support/tickets/:id/messages` | Owner | Reply; `Idempotency-Key`, ticket state guard |
| POST | `/me/support/tickets/:id/close` | Owner | Đóng; `If-Match` |
| POST | `/me/support/tickets/:id/reopen` | Owner | Mở lại trong 7 ngày; `If-Match` |

Review/return/support/report không nhận buyer ID từ body. Attachment dùng upload intent + signed URL ngắn hạn, server kiểm tra MIME/size/quarantine trước khi gắn entity. Request return chỉ chứa item ID/quantity/reason, refund amount do server tính từ snapshot. Các endpoint này theo [Reviews](../domains/reviews.md), [Returns](../domains/returns-refunds.md), [Support](../domains/customer-support.md).

## 8. Notification và campaign (MVP)

| Method | Path | Auth | Quy tắc |
|---|---|---|---|
| GET | `/me/notifications` | Buyer | Cursor/limit, unread count; owner-only |
| POST | `/me/notifications/:id/read` | Owner | Idempotent |
| POST | `/me/notifications/:id/archive` | Owner | Idempotent; ẩn khỏi default inbox, không xóa lịch sử |
| GET/PATCH | `/me/notification-preferences` | Buyer | Category×channel gồm in-app/email/push, marketing mặc định off; `If-Match`, consent version |
| POST/DELETE | `/me/push-subscriptions[/:id]` | Buyer | Endpoint của thiết bị actor; delete/revoke |
| GET | `/me/stream` | Buyer | SSE order/inbox hint, `Last-Event-ID`, heartbeat; không stream PII |
| GET/POST | `/seller/shops/:shopId/notification-proposals` | Shop member | Đề xuất product/promo announcement của shop; capability + MFA |
| GET/POST | `/admin/notification-campaigns` | Admin | List/tạo draft với audience rule, content revision, schedule |
| POST | `/admin/notification-campaigns/:id/submit` | Admin | Gửi duyệt, `If-Match` |
| POST | `/admin/notification-campaigns/:id/decision` | Approver | Approve/reject, second approver cho system critical, reason/audit |
| POST | `/admin/notification-campaigns/:id/pause` | Admin | Ngừng fanout, không thu hồi noti đã gửi |
| GET | `/admin/notification-campaigns/:id/report` | Admin | Aggregate progress/delivery, không buyer PII |

Seller/Admin có inbox và SSE namespace tương ứng (`/seller/shops/:shopId/notifications`, `/admin/notifications`, `/seller/shops/:shopId/stream`, `/admin/stream`), kiểm quyền và shop scope. Campaign body không nhận recipient ID tùy ý; server resolve audience/consent. Nội dung/URL/template validation và state theo [Communications](../domains/communications.md). SSE event là hint (`notification.created`/`order.updated`), không là bản sao dữ liệu order.

## 9. Admin Dashboard (MVP)

Mọi route `/admin/*` cần role/capability phù hợp + MFA + resource policy; mutation có Origin/CSRF, `If-Match` khi ghi đè, audit reason.

| Resource | Endpoint tối thiểu |
|---|---|
| Session/access/user | `GET /admin/session`, `GET /admin/users`, `GET /admin/users/:id`, `POST /admin/users/:id/{suspend,reactivate}`, `POST/DELETE /admin/users/:id/roles[/:role]`, `GET /admin/audit`; capability `users.view/manage` hoặc `roles.grant`, MFA/recent auth, `If-Match`, reason/audit; không tự nâng quyền/khóa chính mình |
| Catalog & CMS | Product/category/variant/media/listing CRUD/review/publish; `GET/POST /admin/cms/pages`, `GET/PATCH /admin/cms/pages/:id`, `POST /admin/cms/pages/:id/{submit,decision,publish,rollback}` với version/reason |
| Localized content | `GET/PUT /admin/catalog/products/:id/translations/:locale`, `GET/PUT /admin/catalog/categories/:id/translations/:locale`, `GET/PUT /admin/cms/pages/:id/translations/:locale`, `POST .../translations/:locale/{submit,decision,publish}`; capability owner, `If-Match`, reason/audit, reviewer khác author theo policy; preview private/noindex |
| Shop onboarding | `GET /admin/shop-applications`, `GET /admin/shop-applications/:id`, `POST /admin/shop-applications/:id/decision`, shop suspend/reactivate với reason |
| Inventory/promotion | Stock adjustment có reason/version; promotion/coupon publish/disable |
| Orders/returns/support | Shop-order transition; `GET /admin/returns`, `POST /admin/returns/:id/decision`; `GET /admin/support/tickets`, `POST /admin/support/tickets/:id/{assign,transition,messages}` |
| Moderation | `GET /admin/reviews/queue`, `POST /admin/reviews/:id/decision`; `GET /admin/moderation/reports`, `POST /admin/moderation/reports/:id/decision`, `POST /admin/moderation/actions/:id/revoke`; product/listing queue; reason, MFA, conflict-of-interest guard |
| Notifications | Campaign draft/approve/schedule/pause/report, inbox |
| Reporting/operations | `GET /admin/reports/overview`, `POST /admin/reports/exports`, `GET /admin/operations/overview`; scoped export có audit/private URL |

Localized content route dùng `:locale=vi|en`. Ví dụ đầy đủ: `PUT /admin/catalog/products/:id/translations/en` tạo/cập nhật draft, `POST /admin/catalog/products/:id/translations/en/submit`, `/decision`, `/publish`; category và CMS page dùng cùng suffix dưới owner path trong bảng. Media alt translation là field của product media revision qua `PUT /admin/catalog/products/:id/media/:mediaId/translations/:locale`. Draft mới dùng `If-None-Match: *`; cập nhật dùng `If-Match`, mọi decision/publish dùng version + reason/audit. Seller shop translation chỉ là draft/proposal và đi qua review policy của `shops`, không tự public. OpenAPI/generator phải khai báo DTO và error cho từng owner trước triển khai.

## 10. Seller API cho khu `/seller/*` của Admin workspace (MVP)

| Method | Path | Auth | Hành vi |
|---|---|---|---|
| GET/POST | `/seller/applications` | Verified buyer + MFA | List/nộp hồ sơ mở shop |
| GET/PATCH | `/seller/applications/:id` | Applicant | Xem/sửa draft/needs_changes, `If-Match` |
| GET | `/seller/shops` | Member | Danh sách shop/capability của actor |
| GET/PATCH | `/seller/shops/:shopId` | Shop member | Profile theo capability/version |
| GET/PUT | `/seller/shops/:shopId/translations/:locale` | `shop.profile.manage` | Draft tên/mô tả/slug/media alt của shop mình, `If-Match`; publish/review theo policy, không sửa locale khác/shop khác |
| GET/POST/DELETE | `/seller/shops/:shopId/members[/:id]` | Owner/manager | Invite/role/revoke, MFA/audit |
| POST | `/seller/invitations/:token/accept` | Verified invitee + MFA | Token one-use/email match; tạo membership idempotent |
| POST | `/seller/shops/:shopId/activate` | Owner + MFA | Profile/media/Terms version/policy hoàn chỉnh, `If-Match`, audit |
| POST | `/seller/shops/:shopId/ownership-transfer` | Current owner + recent auth/MFA | Target member active/MFA; confirm bởi target và audit, không để shop mất owner |
| POST | `/seller/shops/:shopId/ownership-transfers/:id/accept` | Target member + recent auth/MFA | Chỉ pending transfer của chính actor, CAS/version; chuyển owner atomic, giữ ít nhất một owner |
| GET/POST/PATCH | `/seller/shops/:shopId/listings[/:id]` | `listing.manage` | Draft, giá, media, submit review, pause |
| GET/POST | `/seller/shops/:shopId/inventory/adjustments` | `inventory.manage` | Stock/movement có reason/version |
| GET | `/seller/shops/:shopId/orders` | `orders.manage` | Chỉ shop orders của shop |
| POST | `/seller/shops/:shopId/orders/:id/transition` | `orders.manage` | State hợp lệ, reason/version/idempotency |
| GET/POST | `/seller/shops/:shopId/promotions` | `promotions.manage` | Campaign/voucher shop scope |
| GET/POST | `/seller/shops/:shopId/support/tickets` | `support.reply` | Ticket shop scoped, PII redacted |
| POST | `/seller/shops/:shopId/support/tickets/:id/messages` | `support.reply` | Trả lời, không thấy internal note |
| GET | `/seller/shops/:shopId/returns` | `returns.review` | Return thuộc shop |
| POST | `/seller/shops/:shopId/returns/:id/proposal` | `returns.review` | Đề xuất, Admin quyết định cuối |
| GET | `/seller/shops/:shopId/reviews` | `reviews.reply` | Published/pending của shop theo policy |
| POST | `/seller/shops/:shopId/reviews/:id/reply` | `reviews.reply` | Reply chờ moderation |
| GET | `/seller/shops/:shopId/reports/overview` | `analytics.view` | Aggregate shop, asOf/lag |

Server kiểm membership/capability/shop status/resource ownership từng request; cross-shop ID trả 404, suspended shop mutation 423. Seller không sửa canonical product của shop khác, payment outcome hoặc buyer address đầy đủ. Review seller reply và return proposal không được tự phê duyệt; Admin quyết định theo policy và audit.

## 11. F2/F3 và internal event

F2 thêm `/flash-sales` và `/recommendations` read, seller/admin campaign endpoints theo domain spec. F3 có thể thêm `/vN/advertiser/campaigns`, `/vN/admin/ads/*` và public sponsored placement có disclosure; `/vN/affiliate/out/:opaqueCode` chỉ redirect tới **partner domain allowlisted** sau kiểm offer/consent. `GET /vN/external-offers/:id` trả nhãn đối tác, nguồn dữ liệu/giá và thời điểm cập nhật, không trả `listingId` có thể đưa vào Cartasy cart. Ad billing thật (nếu được duyệt) dùng namespace `/vN/advertiser/billing/*` và `/vN/admin/ad-billing/*`, auth/role/provider receipt riêng. Các path ở đây là **định hướng F3**, chưa là contract OpenAPI hay quyền truy cập đã duyệt; không mở route/schema F3 trong F1. [Advertising](../domains/advertising.md), [Affiliate](../domains/affiliate.md), [Ad billing](../domains/advertising-billing.md), [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

Worker đọc outbox trực tiếp bằng database adapter; không mở public webhook mock payment. Provider thật là thay đổi sản phẩm; callback route tương lai phải signature trên raw body, timestamp/replay validation, event receipt/idempotency và ADR riêng theo [Integrations](../platform/external-integrations.md). OpenAPI sinh từ Nest controller/DTO, lưu tại `packages/contracts/openapi/cartasy.v1.json`; `openapi-typescript` sinh type. CI fail nếu artifact/generated type lệch hoặc endpoint không được mô tả.


## 12. Mutation DTO chuẩn cho vertical slice F1

Endpoint inventory ở trên là namespace tối thiểu. Các route chi tiết sau cố định để AI Agent tạo controller/OpenAPI; không tạo route tự phát theo UI. Tất cả mutation nhận JSON object allowlist, không mass-assign ID/owner/status; audit actor/shop từ session và resource. `POST` có side effect retry dùng `Idempotency-Key`; `PATCH/decision` dùng `If-Match` là resource version/ETag hiện tại. File upload dùng upload intent riêng và media ID đã scan, không đưa binary/base64 vào JSON business endpoint.

| Nhóm | Route F1 | Payload chính và guard |
|---|---|---|
| Seller onboarding | `POST /seller/applications`, `POST /seller/applications/:id/submit`, `/seller/shops/:shopId/activate`, `/seller/invitations/:token/accept` | `shopName` 3–120, `slug` unique, `description` ≤2000, contact verified; submit/activate cần recent auth/MFA, version; invite one-use, email match |
| Product proposal | `GET/POST /seller/shops/:shopId/product-proposals`, `GET/PATCH /seller/shops/:shopId/product-proposals/:id`, `POST /seller/shops/:shopId/product-proposals/:id/submit`, `GET /admin/catalog/product-proposals`, `POST /admin/catalog/product-proposals/:id/decision` | Canonical fields, media IDs/provenance; `listing.manage`, `If-Match`; Admin `catalog.review` chọn `matchedProductId` hợp lệ hoặc tạo canonical product qua Catalog owner, reason/audit; seller không mass-assign product status |
| Admin canonical catalog | `GET/POST /admin/catalog/products`, `GET/PATCH /admin/catalog/products/:id`, `POST /admin/catalog/products/:id/decision` | Category/variant/brand/media; `catalog.write/review`, version/reason; product published mới public |
| Admin category/media | `GET/POST /admin/catalog/categories`, `PATCH /admin/catalog/categories/:id`, `POST /media/upload-intents` | Category cycle/slug, media type/size/checksum, scan/quarantine |
| Seller listing | `GET/POST /seller/shops/:shopId/listings`, `GET/PATCH /seller/shops/:shopId/listings/:id`, `POST /seller/shops/:shopId/listings/:id/submit`, `POST /seller/shops/:shopId/listings/:id/pause` | `variantId, priceMinor` string, media IDs, quantity policy; shop active, `listing.manage`, version |
| Admin listing | `GET /admin/listings/queue`, `POST /admin/listings/:id/decision` | Approve/reject/pause, reason/version, shop/product/media eligibility |
| Seller inventory | `GET /seller/shops/:shopId/inventory`, `POST /seller/shops/:shopId/inventory/adjustments` | `listingId, delta` integer, `reasonCode`, version; không trực tiếp set on_hand |
| Promotion | `GET/POST /seller/shops/:shopId/promotions`, `POST /seller/shops/:shopId/promotions/:id/submit`, `GET /admin/promotions/queue`, `POST /admin/promotions/:id/decision` | Scope/eligibility/quota/window, `promotions.manage`/admin approval |
| Shop order | `POST /seller/shops/:shopId/orders/:id/transition` | `nextStatus, reasonCode`, version/key, transition đúng state, không nhận payment result |
| Return/support/review | Seller/admin route ở mục 7/9/10 | Proposal/reply khác final decision; attachment ID private/scan |
| Report/takedown | `POST /reports` và `/admin/moderation/*` | `resourceType/resourceId/reason/evidenceIds`, action gọi use case của owner |

Ví dụ body seller application tạo draft: `{"shopName":"Gian hàng A","slug":"gian-hang-a","description":"Mô tả gian hàng","contactEmail":"seller@example.test"}`. API không nhận `ownerId` hoặc trạng thái `approved`. Admin decision: `{"decision":"approved","reasonCode":"POLICY_PASSED"}`; approval tạo shop/membership một lần. Ví dụ return decision: `{"decision":"approved","reasonCode":"ITEM_ISSUE","restock":false}`; server tính amount từ order/payment snapshot và payment cap, không nhận `refundAmount` từ client.

Campaign draft request gồm `type` (`new_product|promotion|system`), `shopId` chỉ khi seller proposal đúng shop, `title`, `body`, `relativeUrl`, `audienceRule` enum/version và `scheduledAt` UTC. Admin approval/schedule lưu revision/content hash; fanout đọc snapshot audience, vẫn recheck consent tại send. Không nhận `recipientIds` arbitrary hoặc HTML raw. Response tạo resource có `id,status,version,createdAt`; response decision có `id,status,version,auditId`. [Communications](../domains/communications.md) sở hữu policy consent/quiet hours, [data model](data-model.md) sở hữu unique/transaction, [security](security-privacy.md) sở hữu policy.
