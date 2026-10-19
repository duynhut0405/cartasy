---
doc_id: QA-STRATEGY
title: Cartasy Testing and Quality Strategy
status: accepted
owner: Cartasy Engineering & Product
last_updated: 2026-10-19
version: 4.4.0
source_of_truth: Test layer, ma trận rủi ro và release gate
review_cycle: Mỗi milestone và sau sự cố nghiêm trọng
---

# Kiểm thử và chất lượng

## 1. Nguyên tắc

Kiểm tra rule ở tầng thấp nhất chứng minh được, nhưng checkout/payment/order/notification phải có E2E gần production. Không dùng coverage % hoặc test `--passWithNoTests` làm bằng chứng feature hoàn thành. Fixture mô phỏng phải được gắn nhãn, không dùng đánh giá/giao dịch giả làm số liệu thật.

## 2. Các tầng test

| Tầng | Nội dung |
|---|---|
| Static | `format:check` (Prettier + Tailwind class order), ESLint, TypeScript strict, dependency direction/cycle, unused export, secret/dependency scan; Husky/lint-staged chỉ hỗ trợ local, CI là gate |
| Unit | Money integer, price/discount allocation, state machine, authorization policy, quote fingerprint, content validation |
| Integration | PostgreSQL constraint/lock/transaction, reservation, order split, idempotency, outbox/inbox, migration, object storage/SMTP adapter |
| Contract | OpenAPI/controller/DTO/generated client drift, stable error, pagination, auth provider snapshot |
| E2E | Browser Buyer Web và một Admin app với Seller/Platform role F1, PostgreSQL/Mailpit/object storage |
| Security | CSRF/CORS, MFA, object/tenant scope, price tamper, mock scenario isolation, upload/SSRF F2 |
| Performance | Feed/search, quote/checkout concurrent, stock hot listing, outbox/email backlog |
| Recovery | Backup/restore, payment pending replay, worker crash, duplicate event, app rollback |

Test integration dùng PostgreSQL thật cô lập; mock chỉ clock/ID và provider ngoài. Kiểm tra migration từ database sạch và fixture schema trước, không chỉ generate SQL thành công. Payment adapter fake có kết quả deterministic cho success/failure/timeout và event trùng.

## Ma trận nghiệp vụ mở rộng F1

| Module | Ca bắt buộc |
|---|---|
| Identity/session/OTP | Token một lần/hết hạn, revoke, MFA, CSRF, role đổi tức thời, profile/address owner |
| Search/wishlist | Unpublished không lộ, Unicode/facet/cursor, guest merge lặp, cross-user wishlist |
| Promotion/inventory | Quota đồng thời, discount allocation, last-stock race, reservation expiry/commit/release |
| Review | Chỉ order delivered_simulated, một review/item, moderation/XSS, badge simulation |
| Return/refund | Window/quantity, partial/multi-shop refund cap, duplicate decision, restock một lần |
| Support | Ticket/attachment owner, internal note redaction, SLA reminder, state conflict |
| Marketplace F1 | Hai shop và hai buyer, membership revoke, suspend, shop/workspace switch cache isolation, identity có cả Seller/Platform role và deep link cross-role denial |
| Trust/abuse | Report cross-shop, evidence privacy, moderator conflict, takedown race, appeal reviewer độc lập |

Không test chỉ bằng mock repository cho invariant transaction; dùng PostgreSQL thật và fake clock ở worker. Không lấy việc tài liệu có testcase làm bằng chứng đã chạy.

## Notification, CMS và reporting gate MVP

| Luồng | Ca bắt buộc |
|---|---|
| Transactional noti | Mixed-shop order: buyer nhận parent, mỗi seller chỉ shop-order mình; duplicate/retry không nhân đôi |
| Announcement | Product published/seller proposal/Admin approval, consent revoke, quiet hours, frequency cap, campaign pause/restart |
| Web Push/SSE | Permission denied/iOS Home Screen fallback, subscription 410 cleanup, SSE disconnect/replay/cross-user, no PII payload |
| CMS | Draft/review/schedule/publish/rollback, XSS, unpublish shop/product, cache invalidation |
| I18n Web | Route `/vi`/`/en`, locale negotiation/unsupported locale, message key + ICU placeholder parity, localized content/slug uniqueness, unpublished translation 404, locale switch, canonical/hreflang/sitemap, cache/search isolation, translated simulation notice, long text/mobile/a11y |
| Frontend UI/store | shadcn/ui bản Radix + Tailwind token ở Web/Admin, keyboard/focus/contrast; Zustand chỉ giữ UI state dùng chung, reset khi logout/đổi actor/workspace/shop; React Query không lộ cache actor/shop cũ, URL giữ filter/cursor; hydration không đọc server singleton |
| Reporting | Two-shop isolation, projection rebuild/lag/asOf, refund reconciliation, no real GMV label |

## 3. Ma trận luồng trọng yếu

| Luồng | Kiểm chứng tối thiểu |
|---|---|
| Catalog | Published/disabled, variant, listing theo shop, media quyền sử dụng, search/filter/cursor |
| Cart | Guest persist/merge retry; user add/update/remove; price không lấy từ client |
| Quote | N shop, coupon đủ/không đủ điều kiện, phí từng shop, tổng đúng, VND integer |
| Checkout | Đủ hàng, hết hàng, thay giá, cart version cũ, hai request đồng thời cùng/khác key, rollback khi một insert/outbox lỗi |
| Payment | Success/failure/timeout, callback lặp/out-of-order, pending expiry, không có real provider call |
| Inventory | Reserve/commit/release đúng một lần, concurrent hot SKU không âm/oversell |
| Order | Snapshot bất biến, parent/shop state, hủy toàn bộ/từng shop đúng/sai thời điểm, refund mô phỏng không vượt payment, user khác 404 |
| Notification | Email/in-app theo event, retry không gửi trùng logic, link owner-only, nội dung ghi mô phỏng |
| Admin | MFA, role, publish blocker, price/stock audit, state transition, `If-Match` conflict |
| Seller F1 | Hai shop, membership/capability, cross-tenant read/write denied, suspend ngay |
| Privacy | Consent off không chiếu analytics; operational outbox vẫn hoàn tất; export/delete theo policy |

E2E F1 bắt đầu ở mobile viewport: browse/search → variant/listing → guest cart → signup/login → merge → address → coupon → quote → xác nhận mô phỏng → pending → success → order/notification. Thêm đường failure, timeout/retry và cancel. Chạy Chromium và ít nhất một engine khác trước release; accessibility scan kết hợp kiểm tra keyboard thực.

Chạy buyer journey F1 cho `vi` và `en`; đổi locale giữa product/cart không nhân đôi attempt/order, retry noti giữ đúng template locale, VND/tổng tiền không đổi. Kiểm migration backfill `vi`, bản `en` thiếu không bị index và slug redirect giữ đúng locale.

## 4. Dữ liệu thử và nghiên cứu người dùng

Seed dùng catalog demo do dự án sở hữu quyền, hai shop seller độc lập, listing đa variant, một listing stock thấp và promotion hợp lệ/hết hạn. Test factory tạo buyer A/B, shop A/B và admin theo quyền; không dùng PII thật. Clock inject để test expiry và ranh giới ngày; không phụ thuộc thời gian CI. Integration setup/teardown cô lập database hoặc schema.

Trước beta, thử 5–7 người chưa được giải thích sản phẩm. Kịch bản trung tính: tìm sản phẩm, thêm giỏ, hoàn tất đơn, xem notification. Sau đó hỏi họ có bị trừ tiền hoặc có nhận hàng thật không. Ghi kết quả ẩn danh, vấn đề, mức độ và quyết định sửa. Không đóng gate nếu người thử hiểu nhầm ở checkout/order/email hoặc cần người điều phối giải thích.

## 5. Quality gate

Pull request: static + unit/integration liên quan + contract/migration/docs check xanh. Thay đổi checkout, payment, stock, auth, role, promotion, upload hoặc event phải có test race/retry/security tương ứng. Không merge khi OpenAPI, schema, generated client và tài liệu lệch.

Staging release: E2E F1, mock failure/timeout, tenant isolation, HTTPS headers/CORS, backup/restore drill, load test theo [Operations](deployment-operations.md), alert có người nhận, Privacy Policy/Terms và nghiên cứu nhầm lẫn đạt. Production release cần approval, migration/rollback plan và monitoring window. Bug nghiêm trọng có regression test ở tầng thấp nhất tái hiện được.

### Gate riêng cho monetization thật sau MVP

Không gộp vào gate checkout mô phỏng. Ads F3 cần E2E sponsored disclosure/placement/eligibility, seller/admin role, consent, viewability/click dedupe/fraud và serving cap concurrent. Ad billing thật cần provider sandbox, webhook signature/replay, invoice/receipt/credit note/refund/ledger, finance access separation, reconciliation và restore drill; **không** có test nào được chuyển Cartasy `paid_simulated` thành tiền thật. Affiliate cần E2E CTA công bố rời Cartasy, allowlisted redirect/deep link fallback, offer/rights expiry, click consent, partner report validity/reversal/settlement và chứng minh **không có** Cartasy order cho outbound purchase. Revenue report phải tách mock order value, ad revenue và partner commission bằng nguồn chứng từ khác nhau. Mọi đối tác/provider/điều khoản/quota được xác minh lại trước release. [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md), [Advertising](../domains/advertising.md), [Affiliate](../domains/affiliate.md).

## 6. Bằng chứng

Mỗi gate lưu commit SHA, environment, migration version, command, kết quả, timestamp và owner. Checklist chỉ đánh dấu hoàn thành khi đường dẫn tới báo cáo/test hoặc artifact tồn tại. Test flaky được điều tra, không retry vô hạn để tạo màu xanh.
