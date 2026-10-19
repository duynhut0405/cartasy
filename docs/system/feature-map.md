---
doc_id: FEATURE-MAP
title: Cartasy Capability Map
status: accepted
owner: Product & Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Phân loại module MVP và mở rộng
---

# Bản đồ năng lực

| Năng lực | Marketplace MVP | Sau MVP |
|---|---|---|
| Buyer | Auth/profile/address, wishlist, cart, checkout, order, review trải nghiệm mô phỏng, return/support; dùng miễn phí | Follow shop, recently viewed, recommendation, outbound affiliate CTA có công bố, loyalty/creator/live commerce nếu duyệt |
| Ngôn ngữ Web | Storefront `/vi`/`/en` bằng `next-intl`; UI, catalog/shop/CMS, SEO và notification template theo locale, VND giữ nguyên | Locale/thị trường/tiền tệ khác chỉ theo quyết định Product và contract mới |
| Seller | Nhiều shop active, membership/MFA, khu `/seller/*` trong `apps/admin`, listing/stock/price, shop order, basic promo, shop report; self-registration F1 là baseline 4.0 chờ xác nhận phase mới | Bulk tools, advanced analytics, tự tạo ads, ad billing khi đủ gate |
| Admin | Shop/product/media moderation, order/return/support operations, CMS, announcements, RBAC/audit | Advanced risk/fraud/automation |
| Catalog/discovery | Category/product/variant/listing/shop, search/filter/sort, merchandising thủ công | Autocomplete/recently viewed/follow, recommendation cá nhân hóa, ranking nâng cao theo quyết định |
| Giá/khuyến mãi | VND, voucher/campaign cơ bản, shipping fee mô phỏng, snapshot/allocations | Flash sale, bundle, campaign phức tạp |
| Giao dịch | Multi-shop checkout, mock payment/ship/refund, notification, email, Web Push opt-in | Real provider/carrier **chỉ sau quyết định mới** |
| Engagement | Inbox, transactional/announcement campaign, new product/promotion/system notices, preference/consent | Mobile native FCM/APNs, live chat |
| Content & insight | CMS home/banner/static page, buyer funnel, shop aggregate và vận hành | Ads/affiliate attribution, recommendation experiment; báo cáo doanh thu thật tách khỏi mock order |
| Monetization | Không thu Buyer hoặc Seller cho mock order; không bật billing/affiliate outbound F1 | Sponsored Product/Shop/Search/Display có nhãn; quảng cáo thu tiền thật và outbound affiliate/hoa hồng thật chỉ qua gate độc lập |

MVP là **đa seller đang hoạt động**, không chỉ schema “sẵn sàng”. `giá trị đơn mô phỏng`, `chi tiêu quảng cáo thật` và `hoa hồng đối tác đã đối soát` là ba chỉ số khác nhau. [Product scope](product-scope.md) và [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md) sở hữu ranh giới; [domains](../domains/README.md) sở hữu quy tắc từng module. Việc chuyển self-registration từ F1 sang giai đoạn sau chưa được chốt; không tự đổi gate hai seller đang hoạt động.
