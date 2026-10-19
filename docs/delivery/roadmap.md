---
doc_id: PROD-ROADMAP
title: Cartasy Delivery Roadmap
status: accepted
owner: Product & Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Giai đoạn và exit gate
---

# Lộ trình triển khai

Các giai đoạn là kế hoạch, không là bằng chứng code đã hoàn thành. [Checklist](build-checklist.md) lưu trạng thái/bằng chứng; [Audit](audit-and-open-decisions.md) lưu giả định cần xác minh.

## F0 — Nền kiến trúc

Pin stack/lockfile; tạo Web, **Admin chung Seller/Platform**, API, Worker và shared packages; DB/SMTP/object store local; auth/MFA, contract, migration, outbox/job runner, security/CI. Web foundation có `[locale]`, `next-intl`, message parity và token/component structure. Exit: máy mới setup được từ docs, migration sạch/upgrade, contract drift và boundary gate chạy, hai-shop fixture sẵn sàng.

## F1 — Marketplace MVP đa seller, mô phỏng

- Shop onboarding/duyệt, membership/capability, khu Seller `/seller/*` với listing/stock/price/shop order và khu Platform `/platform/*` moderation/operations/CMS/announcement trong cùng `apps/admin`.
- Buyer catalog/search/filter/product/shop, wishlist/cart, promotion/voucher cơ bản, quote/checkout nhiều shop, mock payment/shipping/refund, order history, review trải nghiệm mô phỏng, return/support.
- Web public và buyer journey có `/vi`/`/en`; catalog/shop/CMS translation draft/review/publish theo locale, localized slug/SEO/hreflang/sitemap, API/cache/search và notification template theo locale.
- Inbox, email, Web Push opt-in, notification sự kiện đơn và campaign sản phẩm mới/khuyến mãi/thông báo hệ thống; preference, consent, retry/dedupe và báo cáo delivery.
- Buyer/seller/admin aggregate báo cáo phù hợp quyền; analytics luôn ghi giá trị mô phỏng.
- Exit: hai shop và hai seller hoạt động độc lập trên staging, E2E buyer→seller→platform admin trong cùng Admin app, cross-role/shop denial và toàn bộ gate [Testing](../platform/testing-quality.md)/[Security](../system/security-privacy.md) đạt; không gọi payment/carrier thật.

## F2 — Tăng trưởng marketplace

Flash sale có quota/stock atomic, recommendation rule-based và đo lường, bulk import có quyền, seller reporting nâng cao, dispute/report automation có review. Follow shop/recently viewed/autocomplete/gamification an toàn là đề xuất cần Product xác nhận, không tự thành gate F2. Seller self-registration có thể chuyển sau MVP theo yêu cầu mới, nhưng phase chính thức còn là Open Decision vì baseline F1 đã đặc tả onboarding. Exit: concurrency/load, abuse, fairness, reconciliation và rollback của các tính năng được duyệt được chứng minh. Không đưa flash sale vào F1 bằng cách tái sử dụng voucher thiếu quota.

## F3 — Monetization và hệ sinh thái có điều kiện

Sponsored Product/Shop/Search/Display/Banner có nhãn, advertiser self-service nếu được duyệt; outbound affiliate sang nền tảng đối tác có công bố; recommendation cá nhân hóa khi consent. Đây là **hai chương trình thương mại tương lai tách biệt khỏi mock checkout** theo [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md). Có thể thử placement/đo lường không thu tiền trước; **không ghi doanh thu thật** cho đến khi ad billing hoặc partner commission có chứng từ và đối soát. Pricing model CPC/CPM/fixed, ad payment provider, partner/affiliate API, điều khoản/thuế và thời điểm bật còn mở. Exit cho ad serving: disclosure/eligibility/anti-fraud/tenant/consent; exit **riêng** cho tiền thật: finance/legal sign-off, ledger/invoice/receipt/refund, provider/partner contract, reconciliation, security/incident và test E2E. Creator/live commerce chỉ khi nhu cầu được xác minh.

## Quy tắc release

Mọi giai đoạn có owner, tài liệu nghiệp vụ, API/schema/permissions, test và rollback. SLO/chi phí/provider/domain cụ thể ở [open decisions](audit-and-open-decisions.md) phải được chốt trước public launch; không suy ra chúng đã được duyệt từ kiến trúc đích.
