---
doc_id: DATA-MODEL
title: Cartasy Data Model
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Entity, khóa, invariant, ownership và migration Cartasy
review_cycle: Mỗi thay đổi schema hoặc transaction rule
---

# Mô hình dữ liệu

## 1. Quy ước

PostgreSQL 18 là nguồn giao dịch. Table/column dùng `snake_case`, primary ID UUIDv7 nếu thư viện/runtime được kiểm chứng hoặc UUID random nhất quán; timestamp `timestamptz` UTC. Tiền lưu `bigint amount_minor` và `char(3) currency`, API serialize thành chuỗi; F1 chỉ cho `VND` và 0 phần thập phân. Không dùng float/decimal JS cho tính tiền. Mỗi bảng có module owner duy nhất; FK và transaction bảo vệ invariant ngoài validation ở code.

Auth table do Better Auth adapter sở hữu; migration auth được đưa vào cùng migration stream đã review. `customer_profiles.auth_user_id` là unique FK tới auth user; không copy password/token sang bảng nghiệp vụ. Tên table auth thực tế theo adapter đã pin và được khóa bằng migration/contract test.

## 2. Quan hệ chính

```text
auth user ──1:1── customer_profile ──1:N── addresses
auth user ──N:M── shops (shop_memberships)
shop ──1:N── listings ──N:1── product_variant ──N:1── product
listing ──1:1── inventory_stock ──1:N── inventory_reservations
buyer ──1:1 active── cart ──1:N── cart_items ──N:1── listing
buyer ──1:N── orders ──1:N── shop_orders ──1:N── order_items
order ──1:1── payments ──1:N── payment_attempts
shop_order ──0:1── shipments ──1:N── shipment_events
buyer ──1:N── notifications
shop ──1:N── shop_memberships ──N:1── auth user
notification_campaign ──1:N── campaign_batches ──1:N── notifications
buyer ──1:N── wishlist_items ──N:1── products
order_item ──0:1── reviews
order_item ──1:N── return_items ──N:1── return_requests
buyer ──1:N── support_tickets ──1:N── support_messages
```

## 3. Entity và owner

### Identity và shop

| Bảng | Owner | Field/constraint chính |
|---|---|---|
| Auth user/account/session/verification/2FA | `identity` | Provider ID unique; session/token expiry và revoke; role không đến từ client |
| `customer_profiles` | `customers` | `auth_user_id` unique; display name, preferred UI locale `vi/en`, status |
| `account_data_requests` | `customers` | actor, export/deletion type, state, recent-auth proof reference, legal hold decision, requested/completed timestamps; audit |
| `addresses` | `customers` | `buyer_id`, nhãn, người nhận, phone/address fields, `is_default`; owner-only; soft delete khi đã được snapshot trong order |
| `shops` | `shops` | stable ID, owner, status `draft/active/suspended/closed`, policy version; F1 nhiều seller shop active |
| `shop_translations` | `shops` | unique (`shop_id`, `locale`), unique (`locale`, `slug`); display name/description/SEO fields, revision/publish state; shop identity không đổi theo locale |
| `shop_memberships` | `shops` | unique (`shop_id`, `user_id`); role `owner/manager/staff/analyst`, status, invited_at |
| `shop_capabilities` | `shops` | unique (`shop_id`, `capability`); feature/capability được platform bật cho shop |
| `shop_member_capabilities` | `shops` | unique (`membership_id`, `capability`); quyền thực của member, không suy từ path |
| `shop_applications`, `shop_invitations` | `shops` | applicant/slug/status/version/reviewer; invite token hash/expiry/consumed_at/accepted_user_id, unique business key; activation lưu Terms version/accepted_at/audit reference |
| `shop_ownership_transfers` | `shops` | shop/current owner/target member, status `pending/accepted/cancelled/expired`, version/expiry/recent-auth proof refs; one active transfer/shop; target acceptance atomic với membership role change |

### Catalog và listing

| Bảng | Owner | Field/constraint chính |
|---|---|---|
| `categories`, `brands` | `catalog` | stable ID, status, parent category có cycle guard |
| `category_translations` | `catalog` | unique (`category_id`, `locale`) và (`locale`, `parent_category_id`, `slug`); root category cần partial unique index vì `parent_category_id` NULL; title/SEO, revision/publish state |
| `products` | `catalog` | stable ID, brand/category, status, version, provenance; canonical identity |
| `product_translations` | `catalog` | unique (`product_id`, `locale`) và (`locale`, `slug`); title/description/SEO, revision/publish state |
| `product_variants` | `catalog` | product FK, SKU/option fingerprint unique trong product, status/version |
| `product_proposals` | `catalog` | submitting `shop_id`, author, canonical candidate/versioned payload, media/provenance refs, state `draft/pending_review/needs_changes/approved/rejected`, reviewer/reason/version, optional matched `product_id`; seller không sửa product trực tiếp |
| `product_media` | `media` | product/variant FK, object key, MIME, checksum, rights/provenance, approval status, sort |
| `product_media_translations` | `media` | unique (`media_id`, `locale`); alt text/approved revision cho public locale tương ứng |
| `listings` | `listings` | `shop_id`, `product_id`, `variant_id` composite FK bảo đảm variant thuộc product; price_minor, currency, status/version, purchase limit; unique active (`shop_id`, `variant_id`) |
| `inventory_stocks` | `inventory` | `listing_id` unique, `on_hand >= 0`, `reserved >= 0`, `reserved <= on_hand`, version |
| `inventory_reservations` | `inventory` | order item/listing/qty, status `held/committed/released/expired`, expires_at; một item có tối đa một active reservation |
| `inventory_movements` | `inventory` | listing, delta/loại thay đổi, business reference unique, actor/reason; append-only để đối soát reserve/commit/release/restock/adjustment |

Shop ownership là trên listing. Product canonical có thể được nhiều shop dùng; seller F1 gửi product/variant proposal để moderation thay vì tự sửa canonical content của shop khác. Listing/stock luôn kiểm tra shop scope khi seller đọc/ghi.

### Cart, pricing và promotion

| Bảng | Owner | Field/constraint chính |
|---|---|---|
| `carts` | `carts` | buyer ID, status `active/checked_out`, version; partial unique một cart active/buyer |
| `cart_items` | `carts` | cart ID, listing ID, quantity >0; unique (`cart_id`, `listing_id`) |
| `promotion_campaigns` | `promotions` | shop/global scope, kind, eligibility, start/end, max usage, status, version |
| `coupon_codes` | `promotions` | normalized code unique theo scope, campaign FK, usage limit |
| `promotion_redemptions` | `promotions` | order/buyer/coupon, amount snapshot, status `held/committed/released`; unique business key chống claim lặp |
| `pricing_rules` | `pricing` | versioned shipping/fee policy reference nếu cần; không lưu quote transient làm nguồn giá |
| `flash_sale_campaigns`, `flash_sale_items`, `flash_sale_reservations` | `flash-sales` F2 | listing/shop/window, price/quota, per-buyer quota, held/committed/released/expired; atomic cap |

Quote là read model tạm thời: gồm cart version, listing/stock/promotion policy version, breakdown và expiry. Checkout tính lại trên primary; client chỉ gửi expected total/fingerprint để phát hiện thay đổi. Không tin giá từ cart hoặc quote cũ.

### Order, payment và fulfillment

| Bảng | Owner | Field/constraint chính |
|---|---|---|
| `orders` | `orders` | buyer ID, status, currency, subtotal/discount/shipping/total, address snapshot đã schema-validate, idempotency key, timestamps; unique (`buyer_id`, `idempotency_key`) |
| `shop_orders` | `orders` | order/shop FK, status, subtotal/discount/shipping/total; selected `shipping_method` và `shipping_policy_version` snapshot; unique (`order_id`, `shop_id`) |
| `order_items` | `orders` | shop order/listing/variant reference và immutable snapshot: product/variant/shop, quantity, unit price, item discount, line total |
| `order_status_history` | `orders` | aggregate ID, from/to, actor, reason code, time; append-only |
| `payments` | `payments` | order ID unique, method `mock_card/mock_e_wallet/mock_bank`, amount/currency, paid_minor/refunded_minor, status `pending/paid_simulated/failed/cancelled/partially_refunded_simulated/refunded_simulated` |
| `payment_attempts` | `payments` | payment ID, attempt key unique, scenario, provider event ID unique nullable, status/result, time; không chứa số thẻ/OTP |
| `payment_refunds` | `payments` | payment/shop order, amount_minor, reason, status `completed_simulated`, refund key unique; không gọi provider thật |
| `shipments` | `fulfillment` | shop order ID unique, method giả lập, status, estimated dates; không có carrier/tracking ID thật |
| `shipment_events` | `fulfillment` | shipment ID, state, timestamp, source/actor; append-only |

`orders.status`: `pending_payment → confirmed → partially_fulfilled → completed_simulated`, hoặc `payment_failed/cancelled`. `shop_orders.status`: `pending_payment → confirmed → processing → packed → shipping_simulated → delivered_simulated`, hoặc `cancelled/returned_simulated` theo policy. Parent status được cập nhật trong cùng transaction với child transition và được đối soát từ child; không dùng text tự do. `paid_simulated` chỉ khi có payment attempt success. Không chuyển sang shipping khi payment chưa thành công. “Hoàn tiền” chỉ là trạng thái mô phỏng, không có tiền chuyển về.

### Wishlist, reviews, return và support

| Bảng | Owner | Field/constraint chính |
|---|---|---|
| `wishlist_items` | `wishlist` | buyer/product unique; preferred variant/listing nullable; created_at, sort ID |
| `reviews` | `reviews` | buyer/order_item unique; product/shop snapshot refs, rating 1–5, body, `pending_moderation/published/rejected/hidden`, simulation flag |
| `review_media`, `review_replies` | `reviews` | media đã scan; reply có actor, moderation state và unique policy; report tập trung ở `abuse_reports` |
| `return_requests` | `returns` | buyer/shop_order, reason, state, created/decision deadline, version, actor/reason audit |
| `return_items` | `returns` | request/order_item, quantity >0, refund allocation snapshot, restock flag; tổng pending+approved <= ordered qty bằng transaction |
| `support_tickets` | `support` | buyer, optional order/shop/return reference, assignee, status, SLA deadline, version |
| `support_messages`, `support_attachments` | `support` | ticket/actor, internal-note flag, content sanitized; attachment private/scanned |
| `abuse_reports`, `moderation_actions`, `moderation_appeals` | `risk` | reporter/resource/reason, case state, evidence private, action owner/reviewer conflict guard; appeal review/audit |
| `advertiser_accounts`, `ad_campaigns`, `ad_creatives`, `ad_placements`, `ad_delivery_events` | `advertising` F3 | advertiser/shop scope, slot/creative revision, schedule, disclosure, consent scope, event ID dedupe; không chứa buyer order ID làm chứng từ doanh thu |
| `affiliate_partners`, `external_offers`, `affiliate_links`, `outbound_clicks`, `partner_conversion_reports`, `affiliate_commissions`, `affiliate_reconciliations` | `affiliate` F3 | partner program/rights/offer/destination allowlist, consent-aware click, partner report receipt/reversal/settlement; không FK tới Cartasy order/payment |
| `ad_contracts`, `ad_invoices`, `ad_payment_receipts`, `ad_credit_notes`, `ad_refunds`, `ad_ledger_entries`, `ad_reconciliations` | `ad-billing` sau MVP, có điều kiện | advertiser account, commercial price/currency/tax snapshot, provider receipt unique, immutable accounting entries và reconciliation; không FK tới buyer `payments/payment_refunds` |

Bảng F2/F3 chỉ tạo khi feature được triển khai và gate pháp lý/thương mại tương ứng được duyệt. F1 bắt buộc có shop application/membership/capability và nhiều shop order. Review không chứng minh đã nhận hoặc sử dụng sản phẩm thật. Return/refund và support chỉ tham chiếu tài nguyên buyer/shop hợp lệ; attachment không public. `ExternalOffer` không thể có cart item hoặc order item Cartasy; `affiliate_commissions` không suy từ mock payment. `AdLedgerEntry` chỉ phản ánh hợp đồng/receipt quảng cáo thật theo policy Finance, không dùng `payments` của Buyer. [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

### CMS, notification và reporting

| Bảng | Owner | Field/constraint chính |
|---|---|---|
| `cms_pages`, `cms_blocks`, `cms_revisions` | `cms` | `translation_group_id`, locale, slug, version, approved revision, schedule/publish state; unique (`translation_group_id`, `locale`) và (`locale`, `slug`); block text/media alt nằm trong revision locale tương ứng |
| `notification_campaigns` | `notifications` | creator/shop scope, type, audience rule+version, cutoff, status, approver, schedule, content revision |
| `notification_campaign_batches` | `notifications` | campaign, cursor/range, state/attempt/lease; unique campaign+range, resumable fanout |
| `push_subscriptions` | `communications` | user/device, endpoint hash unique, encrypted endpoint/keys, consent/last_seen/revoked_at |
| `reporting_projections` | `reporting` | shop/date/metric/version/as_of/data_quality, aggregate only; rebuildable, không là tiền thật |
| `report_exports` | `reporting` | requester/filter/reason/job/status/private object key/expiry; authorization recheck trước delivery |

### Platform và tương tác

| Bảng | Owner | Field/constraint chính |
|---|---|---|
| `notifications` | `notifications` | recipient, type, aggregate ID, title/body đã sanitize, read_at, archived_at; unique event/recipient/type |
| `notification_preferences` | `notifications` | actor/category/channel unique, opt-in timestamp/consent version; security/transactional in-app bắt buộc |
| `notification_deliveries` | `communications` | notification/channel/template version, attempts/status; không lưu token email thô |
| `idempotency_requests` | `checkout` | actor, operation, key, request hash, response reference/status, expiry; unique scope/key |
| `outbox_events` | `outbox` | event ID/type/version, aggregate ID, payload tối thiểu, status/attempt/available_at |
| `inbox_events` | `outbox` | consumer + event ID unique để dedupe |
| `background_jobs` | `jobs` | type/version/aggregate reference, available_at, lease/status/attempt/checkpoint; no raw PII |
| `audit_logs` | `audit` | actor, shop scope, action, target, reason, before/after field names/version; append-only |
| `analytics_events` | `analytics` | event ID unique, type/version, occurred_at, consent scope, allowlisted properties |
| `media_uploads` | `media` | uploader/shop, object key, size/MIME/checksum, status, expiry/quarantine |

## 4. Invariant giao dịch

1. `inventory_stocks.on_hand >= reserved >= 0`. Hold: `reserved += qty` nếu `on_hand - reserved >= qty`; commit: `on_hand -= qty; reserved -= qty`; release: chỉ `reserved -= qty`. Transition reservation bằng CAS/status guard, mỗi reservation chỉ kết thúc một lần.
2. Checkout tạo **một** order cho cùng buyer/idempotency key; key khác nhưng cùng cart sau checkout không thể tạo đơn thứ hai từ cart đã đóng. Lock cart và stock rows theo listing ID tăng dần để giảm deadlock. Payment failure là terminal cho order đó; UI có thể đưa item snapshot hợp lệ vào giỏ mới để người dùng thử đặt lại, với key và order mới.
3. Trong cùng transaction checkout: cart lock/close, reservation, order/shop order/items, **shipping method/fee policy snapshot từng shop**, payment pending, coupon redemption hold, idempotency record và outbox. Nếu bất kỳ bước nào lỗi thì rollback tất cả. Shop IDs trong `shippingSelections` khớp chính xác tập shop trong cart; quote fingerprint bao gồm selection/policy version; không tự chọn lại method khi commit.
4. Payment success/failure/expiry khóa payment + order + reservations; kiểm tra transition hợp lệ và commit/release đúng một lần. Event trùng không tạo side effect mới. Mock success tạo shipment mô phỏng cho từng shop order. Hủy shop order sau `paid_simulated` và trước `shipping_simulated` tạo refund mô phỏng theo **snapshot đã phân bổ** cho shop đó, bù stock và hoàn lượt coupon theo policy trong một transaction có audit. `payments.refunded_minor` không vượt `paid_minor`; payment chuyển `partially_refunded_simulated` hoặc `refunded_simulated`. Không có tiền thật được hoàn.
5. `order.total = sum(shop_order.total)`; `shop_order.total = subtotal - discount + shipping`; discount không vượt cơ sở áp dụng; giá trị không âm. Application tính bằng integer và test invariant; DB dùng CHECK/FK/unique khả thi và reconciliation job cho cross-row sum.
6. Order item và địa chỉ snapshot bất biến sau đặt; thay đổi catalog, listing, profile hoặc shop không sửa lịch sử.
7. Order/shop/notification read path lọc buyer hoặc shop owner trên server; FK và application policy chống truy cập ngang tenant.
8. Outbox ghi cùng transaction nguồn; consumer at-least-once/idempotent. Không phát email trực tiếp trong transaction.

## 5. Index và truy vấn chuẩn

- `products(status, published_at DESC, id DESC)`; `product_translations(locale, slug)` unique và search GIN tsvector/trigram theo locale/query thực; `category_translations(locale, parent_category_id, slug)` và `shop_translations(locale, slug)` unique.
- `product_variants(product_id, status)`; unique option fingerprint.
- `listings(product_id, status, price_minor, id)` và `listings(shop_id, status, updated_at DESC)`; unique active shop/variant.
- `inventory_reservations(status, expires_at, id)` cho worker expiry.
- `carts(buyer_id) WHERE status = 'active'`; `cart_items(cart_id, listing_id)` unique.
- `orders(buyer_id, created_at DESC, id DESC)` và `shop_orders(shop_id, created_at DESC, id DESC)`.
- `payment_attempts(payment_id, created_at DESC)`, unique attempt/event key.
- `notifications(recipient_id, created_at DESC, id DESC)`; `wishlist_items(buyer_id, created_at DESC, id DESC)` và unique buyer/product.
- `reviews(product_id, status, created_at DESC, id DESC)`; unique order_item/buyer. `abuse_reports(resource_type, resource_id, status, created_at DESC)` và reporter/status. `moderation_actions(resource_type, resource_id, status)`. `account_data_requests(actor_id, type, created_at DESC)`.
- `return_requests(buyer_id, created_at DESC, id DESC)` và `(status, decision_deadline)`; `support_tickets(buyer_id, updated_at DESC, id DESC)` và admin queue status/SLA.
- `outbox_events(status, available_at, created_at, id)` và `background_jobs(status, available_at, id)` cho `FOR UPDATE SKIP LOCKED`.
- `notification_campaign_batches(campaign_id, state, id)`, unique campaign/range; `notification_deliveries(notification_id, channel, template_version)` unique.
- `push_subscriptions(user_id, revoked_at)` và endpoint hash unique; `shop_applications(status, submitted_at, id)` cho queue Admin.
- `cms_pages(slug, locale)` unique, `cms_blocks(page_id, position)`; `reporting_projections(shop_id, date, metric, version)` unique.

Localized rows chỉ được public khi resource gốc và translation của locale đều published; slug cũ lưu redirect mapping (`owner`, `locale`, `old_slug` → resource ID/current slug) có unique key và chống loop. Việc đổi slug không đổi ID, listing/cart/order snapshot hoặc quyền shop. Cache/search index có locale trong key; chỉ index bản dịch published. Migration F1 backfill text hiện có vào `vi`, tạo constraint/index theo locale, đối chiếu row count/slug collision rồi chuyển read path; **không** tự sinh `en`. Rollback giữ bản `vi` và mapping slug tới khi route cũ đã chuyển, không làm mất order snapshot. [ADR-0012](../decisions/ADR-0012-multilingual-storefront.md).

List dùng cursor/keyset và deterministic tie-breaker ID. Index mới cần query plan trên fixture phân bố gần thực tế; không thêm GIN/index toàn JSONB vì phỏng đoán.

## 6. Vòng đời dữ liệu

Quy trình SQL migration, seed, kiểm drift và recovery thuộc [Database migrations](../platform/database-migrations.md). Retention/ẩn danh PII thuộc [Security](security-privacy.md). Không hard-delete product/listing/shop có order; archive và giữ snapshot bất biến. Search/cache/analytics là projection có thể rebuild, không là nguồn quyết định giá, tồn kho hoặc quyền.
