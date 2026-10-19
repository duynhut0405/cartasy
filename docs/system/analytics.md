---
doc_id: DATA-ANALYTICS
title: Cartasy Analytics and Tracking
status: accepted
owner: Cartasy Product & Data
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Event contract, KPI và ranh giới consent
review_cycle: Khi thay đổi funnel hoặc privacy policy
---

# Analytics

## 1. Phân loại event

**Operational/domain event** như `order.placed`, `payment.mock_succeeded`, `inventory.reservation_released` và `shop_order.status_changed` phục vụ tính đúng, notification và audit; ghi outbox cùng transaction bất kể lựa chọn analytics. **Product analytics event** chỉ chiếu/gửi khi consent hợp lệ và không có GPC/DNT theo policy. Từ chối analytics không làm mất order hoặc notification nghiệp vụ.

Event schema versioned, `eventId` unique, `occurredAt` UTC, `source`, `actorId` nội bộ hoặc anonymous ID khi được phép, `sessionId` khi được phép, `shopId/productId/listingId/orderId` chỉ nếu cần. Không đưa email, địa chỉ, phone, session cookie, full URL hay payment input vào analytics. Event client có thể giả mạo, không làm nguồn cho order/payment/inventory.

Web product analytics có thuộc tính `locale=vi|en` lấy từ route đã validate, chỉ cho event có consent; không gửi full localized URL hoặc bản dịch/query text. Báo cáo có thể phân đoạn theo locale nhưng không cộng trùng hai lần cho cùng order khi người dùng đổi ngôn ngữ; operational event giữ ID/state độc lập locale.

## 2. Event dictionary F1

| Event | Nguồn | Khi phát | Thuộc tính riêng tối thiểu |
|---|---|---|---|
| `search_performed` | Client, consent | Kết quả search hiển thị | resultCount, filterCount; query text không lưu mặc định |
| `product_list_viewed` | Client, consent | List hiển thị | surface, itemCount |
| `product_viewed` | Client, consent | Detail hiển thị | productId |
| `listing_selected` | Client, consent | Chọn shop/variant listing | productId, listingId, shopId |
| `cart_item_added` | Server sau commit và consent | Add user cart | listingId, quantity |
| `guest_cart_item_added` | Client, consent | Add guest local | listingId, quantity |
| `checkout_quote_viewed` | Client, consent | Quote hiển thị | shopCount, itemCount, totalBucket |
| `checkout_submitted` | Server sau commit và consent | Order pending được tạo | orderId, shopCount, itemCount, totalBucket |
| `mock_payment_succeeded` | Server sau settlement và consent | Payment success | orderId, method |
| `mock_payment_failed` | Server sau settlement và consent | Payment failed | orderId, reasonCode |
| `order_cancelled` | Server sau commit và consent | Hủy đơn | orderId, stage |
| `shop_order_cancelled` | Server sau commit và consent | Hủy một shop order | shopOrderId, stage |
| `mock_refund_completed` | Server sau commit và consent | Refund mô phỏng | orderId, shopOrderId, totalBucket |
| `shop_order_status_changed` | Server sau commit và consent | Tiến trình đơn | shopOrderId, status |
| `notification_opened` | Client, consent | User mở notification | type, aggregateKind |
| `simulation_confusion_reported` | User feedback, consent | Người dùng báo hiểu nhầm | surface, reasonCode; không lưu free text trong event |

Không gọi `mock_payment_succeeded` là giao dịch/doanh thu thật. `totalBucket` là khoảng giá, không là dữ liệu thanh toán cá nhân. Operational outbox event có thể chứa ID để đối soát nhưng analytics projection phải qua consent filter; không copy toàn payload.

## 3. Funnel và KPI

Funnel: product list/search → product detail → listing selected → cart add → quote viewed → checkout submitted → mock payment succeeded. Dùng cùng eligible identity và attribution window 24 giờ cho funnel phiên; nếu thiếu consent/identity, chỉ báo số event riêng và độ phủ, không tính conversion rate giả.

| KPI | Định nghĩa |
|---|---|
| Activation | Buyer mới có mock payment success đầu tiên / buyer mới đủ điều kiện trong cohort |
| Checkout completion | Order `paid_simulated` / order được submit, theo cohort/ngày |
| Payment failure rate | Payment failed / payment attempt có outcome; pending báo riêng |
| Cart-to-checkout | Buyer consented submit checkout / buyer consented đã add cart |
| D1/D7 retention | Buyer quay lại ngày 1/7 tính theo `Asia/Ho_Chi_Minh` / buyer mới trong cohort |
| Confusion rate | Feedback báo tưởng thanh toán/giao hàng thật / người tham gia nghiên cứu hoặc active user có feedback |
| Stock conflict rate | Checkout bị `OUT_OF_STOCK` / checkout attempt |
| Notification delivery | Delivery thành công trong ngưỡng / notification được tạo |

Dashboard phải hiển thị nguồn, event version, consent coverage, timezone, cửa sổ attribution, bot filter và ngày cập nhật. Không biến tổng giá trị đơn mô phỏng thành GMV/doanh thu thật. Seller F1 chỉ thấy aggregate theo shop với ngưỡng chống tái nhận dạng; không thấy anonymous/user ID.

Trong F3, event `ad_impression/ad_click/ad_spend_eligible`, `affiliate_outbound_click`, `partner_conversion_reported/reversed`, `ad_invoice_paid/refunded` thuộc namespace và nguồn bằng chứng khác nhau. Impression/click không chứng minh tiền được thu; outbound click không chứng minh partner sale; mock `paid_simulated` không là conversion hoặc doanh thu thật. Dashboard có ba nhóm riêng: **giá trị đơn mô phỏng**, **doanh thu quảng cáo đã đối soát**, **hoa hồng affiliate đã đối soát/đã nhận**; mỗi nhóm có currency, `asOf`, nguồn, trạng thái và reversal. Event client không làm căn cứ invoice/commission. [Reporting](../domains/reporting.md), [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

## 4. Chất lượng và vận hành

Worker chiếu event idempotent theo `eventId`; event sai schema vào failed queue có mã lỗi, không log payload cá nhân. Reconciliation so số order/payment success với operational outbox, tách khỏi số analytics consented. Dashboard cảnh báo chênh lệch, projection lag và tỷ lệ event thiếu field. Thử nghiệm A/B phải có hypothesis, primary/guardrail metric, eligibility, thời gian và stop rule; không thử nghiệm cách làm người dùng nhầm giao dịch mô phỏng. Retention và xóa identifier theo [Security](security-privacy.md).

## Event cho các module F1 mở rộng

Wishlist: `wishlist_added/removed` chỉ gồm product ID và actor pseudonymous khi consent phù hợp. Review: `review_submitted/published` chỉ trạng thái và product/shop ID, không gửi body/rating ở raw client event. Return/support: chỉ event operational aggregate `return_requested/resolved`, `ticket_opened/resolved` với request/ticket ID nội bộ và không có reason/message/attachment/PII. Dashboard doanh số/GMV phải luôn ghi “mô phỏng”; không dùng order value làm doanh thu thật.

## Marketplace và engagement MVP

Bổ sung event `shop_application_submitted/approved`, `seller_listing_published`, `notification_campaign_scheduled/sent`, `push_opted_in/revoked`, `cms_page_published` với event version, source và allowlisted properties. Campaign report định nghĩa eligible audience, delivered per channel, provider accepted (không đồng nghĩa đã đọc), opened/clicked chỉ khi consent và measurement được chứng minh. Mọi buyer/seller report có `asOf` và lag; shop report lọc tenant ở server, không gửi buyer ID/PII cho seller. [Reporting](../domains/reporting.md) sở hữu công thức và reconciliation.
