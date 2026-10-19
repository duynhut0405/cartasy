---
doc_id: SEC-PRIVACY
title: Cartasy Security and Privacy Baseline
status: accepted
owner: Cartasy Engineering & Privacy
last_updated: 2026-10-19
version: 4.2.0
source_of_truth: Auth, phân quyền, bảo vệ nghiệp vụ và vòng đời dữ liệu
review_cycle: Trước release và khi thêm bề mặt dữ liệu/quyền
---

# Bảo mật và quyền riêng tư

## 1. Tài sản và trust boundary

Bảo vệ session/credential, địa chỉ/phone/email, quyền shop/admin, giá/tồn kho/discount, order/payment outcome, catalog/media và secret. Browser, email, upload, file import và mọi tích hợp ngoài là input chưa tin cậy. Mock payment **không** cho phép bỏ qua kiểm soát integrity: outcome vẫn tác động order, tồn kho và notification.

## 2. Authentication

Better Auth self-hosted trên API và PostgreSQL session. F1 email/password cần xác minh email; reset token một lần, hạn ngắn, thu hồi session sau reset. Cookie session host-only trên API origin, `HttpOnly`, `Secure` trên HTTPS, `SameSite=Lax` hoặc chặt hơn theo luồng đã test; buyer idle 14 ngày/absolute 30 ngày; admin/seller idle 8 giờ/absolute 24 giờ; không localStorage. Browser Web/Admin (Admin gồm Seller và Platform) gọi API với `credentials: include`, CORS/trusted origin allowlist và CSRF/Origin defense cho request ghi. Next server không nhận cookie host-only của API nên protected page lấy dữ liệu qua client API, không SSR/cache private response; quyền vẫn kiểm tra tại API. Rate limit dùng gateway hoặc PostgreSQL-backed store dùng chung giữa replica theo IP+route và actor khi có session (không dùng bộ đếm in-memory cho production); lỗi auth/recovery không tiết lộ email tồn tại. OAuth khi mở phải chặn implicit account link theo email và test callback/verified email.

Admin và seller bắt buộc MFA TOTP trước quyền thay đổi catalog/giá/order/quyền; backup code hiển thị một lần, lưu bằng cơ chế không đọc lại được hoặc được security review chấp nhận. Role và session MFA proof đọc lại phía server ở mỗi request nhạy cảm. Better Auth route được mount tại boundary riêng; business route có guard mặc định deny-by-default.

## 3. Authorization

| Actor | Scope |
|---|---|
| Guest | Chỉ public product/listing/shop |
| Buyer | Hồ sơ, địa chỉ, cart, wishlist, order, review, return, support và notification của chính mình |
| Content Admin | Catalog/media theo policy, không thay payment outcome |
| Operations Admin | Listing/stock/promotion/order transition được cấp |
| System Admin | Cấp role/cấu hình, vẫn cần MFA và audit |
| Shop member F1 | Chỉ shop có membership active + capability + resource ownership |

### Ma trận capability tối thiểu MVP

| Action | Buyer | Seller member | Content/Operations Admin | System Admin |
|---|---|---|---|---|
| Profile/cart/order của mình | Own | Own khi mua | Chỉ qua support scope/audit | Break-glass/audit |
| Shop profile/listing/stock/promotion | Không | Đúng shop + capability + MFA cho thao tác nhạy cảm | Duyệt/khóa theo policy | Cấp policy/audit |
| Shop order/support/return | Own buyer view/request | Đúng shop + capability, dữ liệu redacted; return chỉ proposal | Transition/return decision theo role/reason | Break-glass/audit |
| Review/report/takedown | Review/report của mình | Reply review shop, report; không tự moderation | Review/report moderation, conflict guard, reason | Break-glass/audit |
| CMS/platform announcement | Không | Chỉ đề xuất shop campaign | Soạn/duyệt tách người cho critical | Policy/audit |
| Campaign recipient/export/report | Không thấy danh sách người nhận | Chỉ aggregate shop, không buyer PII | Aggregate/export có capability và audit | Break-glass/audit |
| Role/membership grant | Không | Owner/manager đúng shop, không cấp platform role | Không tự cấp role cao hơn | Recent auth + MFA + audit |

**RBAC:** platform role/capability quyết định nhóm hành động `/v1/admin/*`; shop role template và capability quyết định nhóm hành động `/v1/seller/*`. **ABAC:** application còn kiểm actor–resource relationship, `shop_id`, shop/account state, order/listing state, MFA/recent auth và policy của hành động. Quyền từ token/session hoặc `GET /v1/portal/context` chỉ là gợi ý cho UI; application đọc membership/capability hiện hành ở mỗi mutation và các read nhạy cảm. Cùng Admin origin không cấp thêm quyền. Tenant filter phải có ở query/repository; test hai shop/hai buyer, identity có hai vai trò, deep link cross-workspace và stale role. Khi đổi workspace/shop, frontend clear cache/form/stream; API vẫn tự kiểm quyền.

`shopId` trong path/body là đối tượng yêu cầu, không là bằng chứng quyền. Application service kiểm tra actor, shop status, membership, capability và ownership; DB query scope theo shop/user, object-level test với hai buyer và hai shop. Trả 404 cho tài nguyên không thuộc actor nếu cần tránh enumeration. Không cho client mass-assign user, shop, role, price, stock, status, discount hoặc audit actor.

Admin `users.view/manage` và `roles.grant` tách khỏi `catalog.write`/`orders.manage`; suspend/reactivate và role grant/revoke cần MFA, recent auth, reason, version, audit. Không cho tự nâng role, tự khóa account hoặc loại System Admin cuối cùng; revoke session/quyền ngay ở request kế. Seller membership không cấp platform role.

## 4. Bảo vệ checkout/payment/order

- Server tính giá/discount/phí, xác thực quote fingerprint và cart version, khóa stock/cart theo thứ tự ổn định; DB CHECK/FK/unique và transaction bảo vệ oversell.
- Idempotency scope actor+operation+key; cùng key/payload trả cùng kết quả, khác payload conflict. Payment attempt/event ID unique; callback lặp không cập nhật đơn hai lần.
- Mock gateway chỉ là adapter nội bộ. Không route public để chọn `success`, sửa `paid_simulated` hoặc tự gọi callback. Scenario control chỉ local/test với authentication nội bộ và fail-closed khi `APP_ENV` khác.
- Không thu số thẻ, tài khoản ngân hàng, OTP hay ảnh chứng từ. Không dùng logo/URL provider thật để giả như đã thanh toán. Production baseline vẫn dùng mock, không chấp nhận secret payment provider thật.
- Điều chỉnh stock, hủy/hoàn mô phỏng và chuyển trạng thái đơn cần role, state guard, lý do và audit. Outbox/notification không chứa địa chỉ đầy đủ.

## 5. Input, media và nguồn catalog

Boundary HTTP dùng whitelist DTO/validation; output map field rõ. SQL parameterized. Upload chỉ image type cho phép sau MIME sniffing, size/pixel limit, filename/object key do server tạo, quarantine/scan và private bucket; media chỉ public sau duyệt, qua signed URL/CDN policy. Xử lý SVG/HTML thận trọng; không serve upload script trực tiếp trên origin auth. Remote image/feed F2 cần SSRF guard (HTTPS, DNS/IP public sau redirect, timeout/size/redirect cap). CSV export escape formula. Không scrape/tái sử dụng media nếu chưa có quyền.

CSP, HSTS, no-sniff, frame protection, referrer policy và cookie attribute được test trên môi trường HTTPS. XSS trong title/description/template phòng bằng encode/sanitize; không render raw HTML seller gửi. Search/analytics rate limit và abuse filter.

## 6. Dữ liệu cá nhân và consent

| Loại | Ví dụ | Xử lý |
|---|---|---|
| Public | Product/listing/shop published | Cache public theo policy |
| Internal | Draft, stock chính xác, audit, operational metric | Role/shop scope |
| Personal | Email, tên, địa chỉ, phone, order association | TLS, quyền tối thiểu, không log/analytics |
| Secret | Password hash, session token, SMTP/S3 credential | Secret manager, rotation, không xuất ra client |

Người dùng được thông báo địa chỉ chỉ phục vụ mô phỏng và không cần nhập địa chỉ thật; vẫn coi mọi địa chỉ đã nhập là PII. Seller không nhận địa chỉ đầy đủ vì không giao thật. User có thể xem/sửa/xóa địa chỉ chưa dùng; order giữ snapshot có chính sách ẩn danh. Web Push subscription chỉ sau browser permission và opt-in trong app; endpoint/keys encrypted at rest, không là PII để log hoặc chia cho seller. Marketing email/push mặc định off, consent có timestamp/version/source và unsubscribe có hiệu lực trước delivery. System/security notice có thể là bắt buộc trong inbox nhưng push vẫn cần browser permission. Product analytics dựa trên consent; operational order/outbox/notification bắt buộc để cung cấp dịch vụ nhưng có payload tối thiểu. GPC/DNT được xử lý theo privacy policy. Không bán hoặc cấp event cấp người dùng cho shop.

### Retention kỹ thuật mặc định

| Dữ liệu | Mặc định ban đầu |
|---|---|
| Session | Buyer idle 14 ngày/absolute 30 ngày; admin/seller idle 8 giờ/absolute 24 giờ; revoke ngay khi reset/khóa quyền |
| Verification/reset token | Verification 15 phút, reset 30 phút, một lần dùng |
| Guest cart local | 30 ngày không hoạt động |
| Access/error log đã redacted | 30 ngày |
| Raw product analytics | 90 ngày; aggregate không định danh tối đa 13 tháng |
| Notification/in-app delivery | 180 ngày |
| Wishlist | Đến khi buyer xóa/account deletion; guest local 30 ngày |
| Review | Đến khi xóa/ẩn danh theo policy; ảnh/moderation evidence tối đa 12 tháng sau đóng case |
| Return/support và attachment | 12 tháng sau đóng, attachment 90 ngày nếu không có hold; ẩn danh PII theo yêu cầu hợp lệ |
| Orphan upload/quarantine | 24 giờ nếu không gắn entity; rejected media theo review policy |
| Order/payment/audit | 12 tháng trước khi review archive/anonymization; address snapshot PII ẩn danh sau 90 ngày kể từ khi đơn kết thúc nếu không có hold |

Các số này là baseline kỹ thuật, phải được chủ dự án/privacy review đối chiếu nghĩa vụ áp dụng trước public launch. Export/delete account cần state machine, xác thực lại, audit, xóa session/địa chỉ/analytics identifier và ẩn danh phần order/audit được phép; legal hold có owner, lý do và hạn. Không hard-delete order để “xóa tài khoản” nếu phá tính toàn vẹn trước khi có policy được duyệt.

## 7. Logging, secret và incident

Structured log chỉ chứa request ID, route template, status, duration, module, error code, actor/resource ID nội bộ khi cần; không URL query, cookie, body, địa chỉ, email, mock payment payload hoặc stack trace public. Secret ở environment manager, rotation được; `.env.example` không chứa giá trị thật. Dependency/secret scan trong CI và trước release; critical/high có đường triage/owner/hạn, không im lặng bỏ qua.

Sự cố: phân loại mức độ → cô lập route/shop/secret → bảo toàn bằng chứng tối thiểu → thu hồi session/credential → khôi phục → thông báo theo nghĩa vụ áp dụng → post-incident review. Có emergency disable cho listing/shop/promotion/mock settlement; thao tác được audit và có runbook.

## 8. Threat model và release gate

| Bề mặt | Lạm dụng | Control/test bắt buộc |
|---|---|---|
| Auth | Credential stuffing, reset replay, CSRF, admin takeover | Shared rate limit, one-time token, MFA, cookie/CORS/Origin E2E |
| Shop | Cross-tenant read/write, stale role | Membership/capability/resource policy, hai-shop integration test |
| Checkout | Price tamper, oversell, duplicate order | Server recompute, row lock, unique key, concurrent test |
| Mock payment | Client ép success, callback replay, pending vĩnh viễn | Internal-only scenario, event unique, expiry/reconcile/alert |
| Promotion | Coupon reuse, negative total | Transactional usage, cap/unique, boundary test |
| Upload/catalog | XSS, malware, content không quyền, SSRF F2 | MIME sniff, quarantine, provenance, remote-fetch guard |
| Notification | Lộ địa chỉ/order người khác | Recipient scope, template redaction, link auth |
| Campaign/Web Push/SSE | Spam, gửi sai audience, subscription leak, cross-user stream | Admin approval, opt-in/frequency cap, encrypted endpoint, session scope, replay refetch |
| Review/return/support | Review giả, double refund, rò attachment/internal note | Eligibility, moderation, unique/refund cap, private signed URL, role redaction |

Public release cần security review có owner, test các control trên HTTPS staging, restore drill, Privacy Policy/Terms cho mô phỏng, account export/delete và incident contact. Control chỉ có trong tài liệu không được đánh dấu đã triển khai.

## 9. Trust boundary cho monetization tương lai

Đơn Cartasy vẫn mô phỏng và không thu số thẻ/tài khoản/OTP của Buyer. Khi bật ad billing thật, advertiser payment nằm ở `ad-billing` với provider/credential/PCI scope được review riêng; role `ads.manage` không tự có quyền xem/điều chỉnh invoice, refund hoặc ledger. Tách dữ liệu buyer, advertiser, provider receipt và đối soát; MFA/recent auth, dual control theo policy Finance, webhook signature/replay guard, idempotency, audit và reconciliation bắt buộc. Không chuyển cờ môi trường để biến mock `paid_simulated` thành tiền thật. [Ad billing](../domains/advertising-billing.md), [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

Outbound affiliate: destination do partner/offer allowlist và policy server quyết định, không dùng URL query arbitrary; kiểm redirect chain/deep link, chống open redirect/SSRF, affiliate stuffing, bot/self-click, disclosure rõ Buyer rời Cartasy. Không gửi PII/order Cartasy sang đối tác hoặc ghi partner conversion nếu thiếu căn cứ và bằng chứng hợp lệ. Partner credential và report thô chỉ truy cập theo capability, retention/consent/hợp đồng. [Affiliate](../domains/affiliate.md).
