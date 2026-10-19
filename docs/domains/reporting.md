---
doc_id: DOMAIN-REPORTING
title: Reporting and Marketplace Analytics
status: accepted
owner: Analytics & Operations
last_updated: 2026-10-19
version: 4.2.0
source_of_truth: Báo cáo buyer funnel, seller/shop và platform; định nghĩa số liệu
---

# Báo cáo và analytics

MVP có một `apps/admin` với dashboard Platform (catalog/shop/order/notification/support/return/stock) và dashboard Seller shop-scoped (listing views, add-to-cart, order/shop order mô phỏng, stock, coupon). Buyer có lịch sử order, không có báo cáo “chi tiêu thật”. Dữ liệu reporting là projection từ event/DB, có watermark/lag; order/payment/permission authoritative vẫn ở primary. Mọi tổng VND hiển thị **“giá trị đơn mô phỏng”**, không GMV/doanh thu/payout thực. Event dictionary/funnel ở [Analytics](../system/analytics.md).

Định nghĩa chuẩn: `orders_created` đếm order ID sau checkout commit; `mock_paid_orders` đếm payment paid_simulated; `shop_order_value_simulated` cộng shop_order snapshot sau discount+shipping cho paid_simulated trừ refunded_simulated; `return_rate` = approved return items / delivered_simulated items, cùng cohort window; `notification_delivery_rate` = sent/eligible attempted theo channel. Báo cáo ghi timezone, range, filter, sample, refresh watermark, consent coverage; không trộn event client chưa được xác thực với doanh số. Refund partial dùng allocation snapshot, dedupe event ID. Rebuild projection theo batch idempotent từ nguồn, không tự sửa order.

Seller query luôn gắn `shop_id` từ membership, chỉ aggregate shop của mình, không nhận buyer email/address/anonymous ID; áp ngưỡng tối thiểu nhóm nhỏ cho funnel/segment để chống tái nhận dạng. Admin export có capability, mục đích, audit, giới hạn rows, CSV formula escape, signed URL hết hạn; không mở toàn bảng PII. Seller không thấy ranking hoặc dữ liệu shop khác trừ benchmark aggregate được sản phẩm phê duyệt ở F2. Role revoke làm request sau bị từ chối ngay, cache key theo shop/actor.

`GET /v1/seller/shops/:shopId/reports/overview` và `/v1/admin/reports/overview` nhận range tối đa 90 ngày/query, granularity day/week, cursor cho bảng; range quá lớn `422`, projection lag trả `asOf`/`dataQuality`, không bịa số 0 khi chưa xử lý. F2 thêm cohort/retention và campaign attribution; F3 ads/affiliate metrics có version và anti-fraud, nhưng **doanh thu thật** chỉ từ ad-billing/partner settlement đã đối soát, không từ mock order hoặc click. Test two-shop isolation, reconciliation snapshot vs projection, timezone/day boundary, refund duplication, consent, export permission và rebuild.


## Metric dictionary và projection

| Chỉ số | Grain/nguồn | Điều kiện ghi nhận |
|---|---|---|
| `orders_created` | Order ID, checkout committed | Một lần/order, kể cả payment failed, tách status |
| `mock_paid_orders` | Payment ID, status transition | Một lần khi `paid_simulated`, không tính retry |
| `shop_order_value_simulated` | Shop order + payment/refund ledger | Tổng snapshot sau discount/phí, trừ refund theo đúng shop |
| `listing_views` | Event client consented | Event ID unique, bot/self-preview loại theo policy; không là số bán |
| `add_to_cart` | Command server accepted | Một event/command ID, không dùng click đơn thuần |
| `return_rate` | Delivered item cohort | Approved quantity / delivered quantity, công bố mẫu và kỳ đo |
| `notification_delivery_rate` | Delivery attempt/channel | Provider accepted / eligible attempted, không khẳng định người dùng đã đọc |

Projection chạy từ outbox/event ID và snapshot authoritative, có `projectionVersion`, `asOf`, `lagSeconds` và `dataQuality` (`ready/rebuilding/stale`). Rebuild dùng shadow version rồi swap sau đối soát; khi stale, API vẫn trả số cũ có nhãn hoặc `503 SERVICE_NOT_READY` theo mức độ, không trả zero giả. Reconciliation mỗi ngày so source order/payment/refund và projection theo shop/date; lệch tạo alert/ticket, không sửa sổ nguồn qua dashboard. Client không dùng báo cáo làm căn cứ checkout/quyền.

## Contract và quyền

`GET /v1/seller/shops/:shopId/reports/overview?from&to&granularity` kiểm membership active + `analytics.view` trên mỗi request; response chỉ aggregate shop, không buyer identifiers. `GET /v1/admin/reports/overview` cần `reports.view` + MFA; export `POST /v1/admin/reports/exports` cần `reports.export`, reason và giới hạn dữ liệu, job tạo private expiring URL, audit actor/filter. `from/to` ISO date, `from <= to` và range tối đa 90 ngày; granularity `day/week`. Cursor cho bảng lớn gắn filter/version. `401` thiếu session, `403` capability, `404` shop khác, `422` invalid range, `503` projection chưa sẵn. Seller/Admin UI có filter, mốc làm mới và disclaimer “giá trị đơn mô phỏng”; không gộp `orders_created` với `mock_paid_orders` thành “doanh thu”.

Test thêm event out-of-order, projection crash/rebuild/replay, order cancellation/refund partial, timezone date boundary, user revoke trong khi export job đang chạy, k-anonymity khi nhóm nhỏ, CSV formula injection và so khớp per-shop vs platform total.

## Ranh giới báo cáo monetization sau MVP

Ad dashboard tách `impressions/clicks/eligible spend` (đo lường), `invoice issued`, `cash received`, `refund/credit note` và `recognized ad revenue` (chỉ sau Finance policy/reconciliation). Affiliate dashboard tách outbound clicks, partner-reported conversions, approved commission, reversed commission và cash received. Mọi con số có partner/provider, kỳ, currency, `asOf`, trạng thái đối soát và quyền xem; chỉ hiển thị aggregate Seller đúng shop/advertiser account. Không cộng mock `shop_order_value_simulated` vào doanh thu quảng cáo/affiliate; không dùng paid_simulated làm bằng chứng partner purchase. [Advertising](advertising.md), [Ad billing](advertising-billing.md), [Affiliate](affiliate.md).
