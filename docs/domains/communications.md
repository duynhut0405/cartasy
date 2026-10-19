---
doc_id: DOMAIN-COMMUNICATIONS
title: Notifications, Email, Web Push and Campaigns
status: accepted
owner: Notifications & Communications
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Quy tắc nghiệp vụ notification, preference, campaign và delivery
---

# Notification, email, Web Push và campaign

## 1. Ranh giới module

`notifications` sở hữu inbox, preference, campaign/audience và recipient dedupe. `communications` sở hữu template, kênh email/Web Push, attempt/retry/dead-letter. `identity` sở hữu token xác minh/reset; communications chỉ chuyển email, không xác thực token. `outbox` sở hữu event delivery sau commit. API/Worker chỉ là adapter; không tự thêm notification trực tiếp từ controller hoặc UI. Dữ liệu chuẩn ở [Data model](../system/data-model.md), HTTP shape ở [API contracts](../system/api-contracts.md).

## 2. Taxonomy và kênh

| Loại | Ví dụ trigger | Người nhận | Inbox | Email | Web Push |
|---|---|---|---|---|---|
| Security bắt buộc | Verify/reset, đăng nhập đáng ngờ, thay quyền | Identity liên quan | Tùy event | Bắt buộc | Không mặc định |
| Transactional bắt buộc | Order placed, mock paid/failed, shipping simulated, cancel/refund/return, support reply | Buyer; seller của đúng shop order; admin queue theo capability | Bắt buộc | Theo severity/template | Opt-in |
| Marketplace hoạt động | Shop application/approval/suspension, listing moderation, low stock | Seller/admin đúng scope | Bắt buộc | Theo policy | Opt-in |
| Marketing tùy chọn | Sản phẩm mới, voucher/promotion, flash sale F2 | Buyer đã opt-in, segmentation hợp lệ | Theo preference | Opt-in riêng | Opt-in riêng |
| System announcement | Tính năng mới, bảo trì, thay đổi chính sách | Audience do Admin duyệt | Bắt buộc nếu là thông báo dịch vụ quan trọng, còn lại opt-in | Theo mức quan trọng/consent | Opt-in |

Không gửi thông báo “đơn đang giao” khi chưa có transition `shipping_simulated` thực sự. Nhãn push/email/inbox đều ghi “mô phỏng” cho payment/ship/refund. Product mới chỉ phát khi product + ít nhất một listing/shop đủ điều kiện public; không tự gửi email cho mọi buyer khi seller lưu draft. System feature announcement chỉ sau khi feature flag/release đã bật đúng audience, không dựa trên merge code.

## 3. Luồng transactional

Business use case chuyển trạng thái trong transaction và ghi outbox event versioned cùng commit. Worker lấy event, resolve recipient theo resource owner/capability, tạo `notification`/`delivery` với unique key `eventId+recipientId+templateKey+channel`, gửi email/push ngoài transaction. Order/shop event có aggregate ID, shop ID, correlation ID, nhưng payload không chứa địa chỉ đầy đủ, token hoặc PII. Seller chỉ nhận shop-order của shop mình; buyer chỉ nhận order của mình. Event xử lý lặp tạo **một inbox/delivery intent logic**; SMTP/Web Push bên ngoài vẫn có thể nhận trùng hiếm gặp nếu worker crash sau provider accept nhưng trước khi lưu receipt. Retry/observability và copy phải chấp nhận giới hạn này, không tuyên bố exactly-once external delivery. Notification thất bại không rollback order.

### Ma trận trigger → recipient → delivery F1

Đây là **contract nghiệp vụ của các transition đã được duyệt**; event key/template key dưới đây là tên logic để viết DTO/OpenAPI/test trước khi code, chưa phải event schema đã triển khai. `I` = inbox + SSE hint khi tab mở; `E` = email theo template/policy; `P` = Web Push chỉ khi user opt-in và browser còn subscription. Mọi hàng `I` vẫn lưu inbox khi tab đóng; Worker retry `E/P` và fallback theo §6–7, không phụ thuộc SSE. Link là đường dẫn nội bộ của app tương ứng, được kiểm lại quyền khi mở; không nhúng dữ liệu order/PII vào push. Các status shipping bên dưới chỉ phát **sau commit transition tương ứng**; nhánh giao thất bại chưa có transition được duyệt.

| Trigger logic / điều kiện | Recipient được resolve tại thời điểm gửi | Kênh và template/content tối thiểu | Deep link |
|---|---|---|---|
| `order.placed` sau checkout commit, payment pending | Buyer owner; seller **chưa** nhận lệnh xử lý trước paid | I; `buyer.order_placed` — mã đơn, “đơn mô phỏng, chờ thanh toán giả lập” | Web `/orders/:orderId` |
| `payment.mock_succeeded` sau paid và shop order confirmed | Buyer owner; từng seller của shop order đã confirmed và còn quyền | Buyer I+E+P `buyer.mock_payment_succeeded`; seller I+E `seller.new_shop_order`; ghi “mô phỏng” | Web `/orders/:orderId`; Admin workspace Seller `/seller/shops/:shopId/orders/:shopOrderId` |
| `payment.mock_failed` hoặc timeout được chốt failed | Buyer owner; seller không nhận order để xử lý | I+E+P `buyer.mock_payment_failed`; nêu retry/đặt lại theo policy, không nói đã trừ tiền | Web `/orders/:orderId` |
| `shop_order.processing` hoặc `shop_order.packed` sau seller/Admin transition hợp lệ | Buyer owner của parent order; seller actor có sẵn state ở portal, không tự gửi lại cho chính actor | I+P `buyer.order_processing`/`buyer.order_packed`; nêu shop và trạng thái mô phỏng | Web `/orders/:orderId` |
| `shipment.shipping_simulated` sau shipment transition | Buyer owner; seller đúng shop nếu transition do worker/Admin và cần xem | Buyer I+P `buyer.shipping_simulated`; seller I `seller.shipping_updated`; ghi “đang vận chuyển mô phỏng” | Web `/orders/:orderId`; Seller `/seller/shops/:shopId/orders/:shopOrderId` |
| `shipment.delivered_simulated` sau shipment transition | Buyer owner; seller đúng shop nếu transition do worker/Admin | Buyer I+P `buyer.delivered_simulated`; seller I `seller.delivered_simulated`; không khẳng định giao hàng thật | Web `/orders/:orderId`; Seller `/seller/shops/:shopId/orders/:shopOrderId` |
| `shop_order.cancelled` hoặc parent cancelled sau policy/compensation commit | Buyer owner; seller đúng shop nếu không phải actor hủy | Buyer I+E+P `buyer.order_cancelled`; seller I `seller.order_cancelled`; nêu shop/phần đơn và hoàn tiền **mô phỏng** nếu có | Web `/orders/:orderId`; Seller `/seller/shops/:shopId/orders/:shopOrderId` |
| `return.requested`/`return.decided`/`refund.mock_completed` sau state tương ứng | Buyer owner; seller đúng shop khi cần xử lý; Admin queue theo capability khi cần duyệt ngoại lệ | I; E cho quyết định cuối/hoàn tiền; P opt-in cho Buyer; template chứa trạng thái và từ “mô phỏng” | Web `/returns` hoặc `/orders/:orderId`; Seller `/seller/shops/:shopId/returns`; Platform `/platform/returns` |
| `shop.application_submitted`/`shop.application_decided`/`shop.activated` | Admin review queue theo capability; applicant hoặc seller được duyệt | Admin I+E khi có action; applicant/seller I+E khi quyết định/activate; không gửi shop khác | Platform `/platform/shop-applications`; applicant `/seller/applications/:id`; Seller `/seller/shops/:shopId` sau khi đã có quyền |
| `listing.proposal_submitted`/`listing.proposal_decided` hoặc `listing.moderation_decided` | Admin moderation queue; seller của đúng shop | I+E khi cần action hoặc có quyết định; không lộ lý do moderation cho Buyer | Platform `/platform/catalog/products`; Seller `/seller/shops/:shopId/catalog/proposals` hoặc `/seller/shops/:shopId/listings` |
| `support.reply_added`/`support.ticket_assigned` | Buyer hoặc seller tham gia ticket; Admin assignee theo capability | I; E theo preference/severity; P cho Buyer opt-in; excerpt phải redacted | Web `/support/:ticketId`; Seller `/seller/shops/:shopId/support`; Platform `/platform/support` |
| `security.role_changed`/`security.suspicious_login` | Identity bị tác động; Admin security queue nếu policy yêu cầu | E bắt buộc, I nếu session còn hợp lệ; không gửi P mặc định; link đến account/security | Route account/security được app guide chốt trong DTO |

`order.placed` và payment event có thể gần nhau nhưng **không gửi seller.new_shop_order** cho pending/failed payment. Từng shop order tạo một notification seller riêng, khóa bằng `eventId+shopId+recipientId+templateKey`; buyer event cùng parent cần dedupe khi nhiều child chuyển cùng batch. Sau role revoke, shop suspend hoặc account deactivate, delivery worker phải kiểm lại policy/resource; notification bắt buộc của Buyer vẫn được lưu theo ownership và retention. Admin queue không là broadcast cho mọi Admin; chỉ user/capability được phân công. `E/P` có thể thiếu theo policy hoặc opt-in mà không làm mất `I`; security email theo auth policy riêng. Nội dung chính xác, locale, HTML/email subject, route account và mức severity cần được chốt trong template/DTO của vertical slice trước gate.

Campaign tùy chọn dùng một luồng khác: `campaign.approved_scheduled → batch fanout`, snapshot audience rồi kiểm lại consent, eligibility, quiet hours/frequency cap trước từng recipient. Template sản phẩm mới chỉ gửi khi product+listing đã public; promotion/voucher chỉ gửi khi active và còn eligibility; maintenance critical cần second approver, audience/capability được duyệt. `campaignId+recipientId+channel` chống gửi logic trùng; deep link public đến product/shop/campaign hoặc trang thông báo hệ thống trong origin, mỗi link qua allowlist. **Follower-specific** trigger/recipient chưa thuộc F1 vì follow shop chưa có data/API; không nhận diện follower từ wishlist hay lịch sử order. Xem §4 và [Open decisions](../delivery/audit-and-open-decisions.md).

F1 có template `vi`/`en`: worker chọn locale từ preference đã lưu của recipient khi tạo delivery intent, lưu `locale` + `templateVersion` để retry ổn định; fallback `vi` nếu preference không hợp lệ/thiếu template transactional, ghi metric. Marketing thiếu bản dịch đã duyệt không được gửi. Link Buyer dùng `/{locale}/...` và ID tài nguyên, Seller/Platform giữ route Admin theo quyền; không nhúng PII vào push. [ADR-0012](../decisions/ADR-0012-multilingual-storefront.md).

## 4. Campaign: sản phẩm mới, khuyến mãi, hệ thống

Admin có capability `notification.campaign.manage` tạo draft; system announcement quan trọng cần second approver khác người tạo. Seller F1 có thể đề xuất campaign sản phẩm mới/promotion **cho shop mình**; admin duyệt trước publish để tránh spam, seller không chọn “all users” hoặc sửa audience trực tiếp. Trạng thái `draft → pending_review → approved → scheduled → sending → completed`; nhánh `rejected/cancelled/paused`. Chỉnh nội dung/audience sau approved trả về draft và cần duyệt lại. Schedule UTC, local preview Asia/Ho_Chi_Minh; quiet hours marketing mặc định kỹ thuật 22:00–08:00, transactional/security không trì hoãn. Giờ yên lặng và cách phân loại campaign cần Product/Privacy duyệt trước public launch theo [audit](../delivery/audit-and-open-decisions.md).

Audience snapshot tại `approved/scheduled` gồm rule version, cutoff time, consent/preference và resource eligibility; trước từng delivery vẫn kiểm tra unsubscribe/active account/suspended shop. Marketing có frequency cap mặc định kỹ thuật 1 push/ngày, 2 email/tuần/buyer (cần Product/Privacy duyệt trước public launch) và dedupe `campaignId+recipientId+channel`. Campaign không lấy order/PII raw để target trong MVP. **Audience có thể thực thi ở F1 chỉ là buyer đã opt-in cho category/channel và các điều kiện account/resource do Admin duyệt.** Category interest hoặc shop follow chỉ được dùng khi feature, consent và API/data riêng được triển khai sau; không suy follow từ wishlist/order. Vì vậy kịch bản “shop mới gửi cho người theo dõi” chưa thuộc F1 khả thi dù campaign sản phẩm mới cho opt-in chung đã có. Không tạo “gợi ý cá nhân hóa” từ lịch sử mua nếu chưa có consent và spec recommendation. Fanout theo batch 500 recipient keyset, checkpoint/cursor, idempotent restart; rate limit provider và global kill switch. Chỉ lưu aggregate delivery report cho seller, không lộ danh sách buyer.

## 5. Inbox, preference và trạng thái

Inbox record `unread → read → archived`; `read_at` một lần, archive vẫn có thể xem theo policy retention. List cursor `created_at,id`; unread count có projection rebuild. Link chỉ tới URL nội bộ allowlist, resource được API kiểm lại sau click. Buyer/seller/admin có inbox tách scope; một identity đổi shop phải clear cache, query key gồm actor/shop. Preference theo `category×channel`: security/transactional in-app bắt buộc; marketing email/push mặc định **off**, chỉ bật bằng opt-in có timestamp/source/consent version; user có thể tắt ngay. Nếu trình duyệt thu hồi permission hoặc user logout, subscription được vô hiệu hóa/không gửi. Email unsubscribe an toàn cho marketing, không đổi preference bảo mật.

## 6. Web Push và realtime

Web đăng ký Service Worker và PushSubscription qua HTTPS sau **hành động của người dùng**; không prompt ngay khi load trang. API lưu subscription endpoint/keys encrypted at rest, gắn user+device/browser, unique endpoint hash, last_seen/expiry, rotate và revoke. Worker dùng VAPID + `web-push`, payload tối thiểu `notificationId/type/title/relativeUrl`, không địa chỉ/tổng tiền chi tiết/OTP; 404/410 xóa subscription, 429/5xx retry có giới hạn. Nếu không hỗ trợ/từ chối permission, inbox + email vẫn dùng được. iOS yêu cầu web app được thêm Home Screen cho push; UI giải thích điều kiện này thay vì hứa mọi tab iPhone đều nhận.

Khi tab đang mở, SSE `/v1/me/stream` gửi hint `notification.created/order.updated` với ID/cursor, rồi client refetch API; không dùng stream làm nguồn dữ liệu. SSE auth cookie, CORS credential/origin allowlist, heartbeat, connection cap, disconnect cleanup, replay cursor; fallback React Query polling. Không mở WebSocket chỉ để gửi noti; WebSocket là quyết định sau này cho live chat hai chiều. Ranh giới hạ tầng ở [Integrations](../platform/external-integrations.md).

## 7. Email và auth OTP

Nodemailer SMTP; Mailpit local, staging mailbox test, production SMTP provider đã duyệt. Template `vi`/`en` versioned text+HTML, escape biến, link allowlist, preview trên Admin; subject transactional nêu mô phỏng. Delivery `queued → sending → sent` hoặc `retry_wait → dead_letter`; exponential backoff+jitter, tối đa 8 lần/24 giờ cho lỗi tạm, lỗi permanent dừng. Không lưu raw reset/verification token trong outbox/log; Better Auth hook tạo token một lần/TTL, gửi qua auth email adapter theo contract provider, resend có rate limit và không tiết lộ email tồn tại. OTP ngân hàng không tồn tại.

## 8. Quyền, validation, lỗi và API

Buyer chỉ list/read/preference/subscription của mình. Seller chỉ inbox/campaign shop đang active với capability; admin campaign quản lý/duyệt/audit theo role, không gửi arbitrary HTML. Campaign title 3–120 ký tự, body 10–500, template key allowlist, relative link cùng origin, image phải media approved; schedule sau hiện tại và trước expiry campaign. Không cho raw HTML/script, URL ngoài allowlist hoặc recipient ID list tùy ý. `401` session hết, `403` thiếu capability/consent, `404` tài nguyên khác owner, `409` stale version/state/campaign duplicate, `422` payload/audience invalid, `429` rate/frequency cap, `503` provider unavailable (notification vẫn giữ retry). Endpoint chi tiết tại [API contract](../system/api-contracts.md).

## 9. Kiểm thử và quan sát

Test recipient matrix order nhiều shop, seller revoke giữa event và delivery, duplicate event/batch/restart, consent revoke ngay trước send, product unpublish trước campaign, quiet hours, frequency cap, XSS/link spoof, Web Push 410, browser permission denied/iOS fallback, SSE reconnect/replay/cross-user, SMTP retry/dead-letter. Metric: event-to-inbox p95, email/push success/failure, queue age, campaign progress, opt-out, provider 429, SSE active connections. Alert backlog/dead-letter và campaign fanout mắc kẹt; vận hành replay theo event/campaign ID có audit.
