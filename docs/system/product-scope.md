---
doc_id: PROD-SPEC
title: Cartasy — Product Scope
status: accepted
owner: Product & Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Mục tiêu, ranh giới mô phỏng và phạm vi phát hành
---

# Đặc tả sản phẩm Cartasy

## 1. Quyết định đã xác nhận

- Cartasy là **marketplace đa nhà bán ngay trong MVP**. Buyer có thể chọn listing từ nhiều shop, đặt một checkout và theo dõi từng shop order. Seller và nhân sự nền tảng dùng **một `apps/admin`** với route và quyền riêng; Seller self-registration còn là [Open Decision M-01](../delivery/audit-and-open-decisions.md).
- Buyer dùng luồng mua sắm Cartasy **miễn phí**. Trải nghiệm đơn hàng, thông báo và hậu mãi giống thương mại điện tử thực tế, nhưng **thanh toán, giao vận và hoàn tiền của đơn Cartasy là mô phỏng**: không thu/chuyển tiền, không giao hàng thật. Giá dùng VND, không có credit/điểm/ví làm phương tiện thanh toán.
- F1 không tích hợp cổng thanh toán, ngân hàng, carrier, payout hoặc quảng cáo tính phí thật. Mọi bề mặt UI/email/push về đơn Cartasy ghi rõ “mô phỏng”.
- Storefront F1 có `/vi` và `/en` bằng `next-intl`, default tiếng Việt; bản dịch UI và nội dung public được duyệt theo locale. Thị trường đầu vẫn Việt Nam, giá/quote/order VND và mô phỏng. [ADR-0012](../decisions/ADR-0012-multilingual-storefront.md).
- Định hướng **sau MVP**: Seller/Advertiser có thể trả tiền thật cho quảng cáo Cartasy; Cartasy có thể nhận hoa hồng affiliate thật khi Buyer bấm liên kết được công bố và mua **trên đối tác**. Các luồng này không tạo/chuyển trạng thái mock order, không biến giá trị đơn mô phỏng thành doanh thu thật. Việc bật phụ thuộc hợp đồng, chính sách, pháp lý/thuế, đối soát và gate riêng theo [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).
- Tài liệu là kiến trúc đích; không ngụ ý ứng dụng đã được cài đặt.

## 2. Persona và vai trò

| Persona | Nhu cầu và phạm vi |
|---|---|
| Guest | Duyệt catalog/shop, tìm kiếm/lọc, guest cart/wishlist cục bộ |
| Buyer | Identity/profile/address, cart/checkout, order, noti, review trải nghiệm mô phỏng, return/refund mô phỏng, support |
| Seller applicant | Đăng ký shop, bổ sung hồ sơ, theo dõi quyết định |
| Shop owner/manager/staff/analyst | Khu `/seller/*` của Admin workspace theo capability và shop scope; listing, stock, promotion, shop order, support, báo cáo |
| Content/Operations/System Admin | Quản trị catalog, moderation, shop onboarding, đơn/return/support, cấu hình, audit và quyền |

Một người có thể đồng thời là buyer và thành viên nhiều shop. Quyền seller được tính từ membership + capability + trạng thái shop + resource ownership ở server.

## 3. Mô hình thương mại bắt buộc

`Product` là nội dung chuẩn; `Variant` là SKU/options; `Listing` thuộc một shop cho một variant và sở hữu giá VND, trạng thái, giới hạn mua. Nhiều shop có thể bán cùng product/variant. `CartItem` tham chiếu listing. Checkout tạo `Order` của buyer với một `ShopOrder`/shop và item/giá/discount/phí/địa chỉ snapshot bất biến. `Payment` và `Shipment` có state mô phỏng. Seller nhìn/điều khiển phần shop order được cấp quyền, không sửa payment outcome. Schema và invariant tại [Data model](data-model.md).

Ba kết quả thương mại có nguồn chứng cứ riêng: **(a)** `MockOrder` từ Cartasy checkout, luôn mô phỏng; **(b)** hợp đồng/hóa đơn/receipt quảng cáo thật nếu sau này bật `ad-billing`; **(c)** outbound click và hoa hồng affiliate chỉ khi đối tác xác nhận. External affiliate offer không là `Listing` có thể thêm vào Cartasy cart; Buyer phải thấy rõ sẽ rời Cartasy để mua thật và giao dịch thuộc đối tác. [Advertising](../domains/advertising.md), [Ad billing](../domains/advertising-billing.md), [Affiliate](../domains/affiliate.md).

## 4. MVP: hành trình end-to-end

1. Seller verified nộp đơn mở shop → Admin duyệt → Seller hoàn tất hồ sơ, tạo listing/giá/stock cho product được duyệt → gửi publish theo policy. Ít nhất **hai shop độc lập** được hỗ trợ trong dữ liệu, API và UI.
2. Buyer/guest khám phá home/category/search/product/shop, chọn variant và listing, lưu wishlist, thêm giỏ. Search/price/stock hiển thị từ dữ liệu đủ điều kiện public; checkout đọc nguồn authoritative.
3. Buyer đăng nhập, chọn địa chỉ mô phỏng, shipping method **cho từng shop**, voucher và xem quote theo từng shop. Xác nhận “không thu tiền/không giao hàng” trước khi đặt.
4. Checkout idempotent giữ stock/voucher, tạo order/shop order/payment pending trong một transaction. Worker xử lý mock success/failure/timeout; không gọi provider thật. Order timeline và notification/email/web push opt-in phản ánh kết quả.
5. Seller xử lý shop order mô phỏng theo state được cấp; buyer xem/hủy đúng quyền. Return/refund mô phỏng, review **trải nghiệm mua sắm mô phỏng** và support ticket hoạt động; không tuyên bố buyer đã nhận/dùng sản phẩm thật.
6. Admin có catalog/shop/moderation/inventory/promotion/order/return/support/CMS/announcement/audit/operations. Báo cáo luôn ghi “giá trị đơn mô phỏng”, không là doanh thu thật.

Năng lực và độ sâu release gate ở [Feature map](feature-map.md), [Roadmap](../delivery/roadmap.md), [Build checklist](../delivery/build-checklist.md).

## 5. Năng lực MVP theo context

| Context | Kết quả bắt buộc |
|---|---|
| Identity & access | Email/password, verification/reset, session revoke, MFA admin/seller, RBAC/capability và object-level policy |
| Catalog & discovery | Product/variant/category/media/listing, search/filter/sort, shop page, moderation, provenance |
| Buying | Wishlist, guest/server cart merge, quote, promotion/voucher cơ bản, multi-shop checkout |
| Transaction | Mock payment success/failure/timeout, reservation, order/shop order, shipping/refund mô phỏng và notification |
| Marketplace | Shop application/review/activation/suspension, membership/invite, khu Seller trong `apps/admin` quản lý listing/stock/order |
| Engagement | Inbox, transactional email, Web Push opt-in, thông báo sản phẩm mới/khuyến mãi/tính năng/hệ thống có phân khúc và preference |
| Trust & service | Review trải nghiệm mô phỏng, return/refund mô phỏng, support ticket, report/takedown/appeal, moderation/audit |
| Content & insight | CMS home/banner/page có publish schedule; báo cáo buyer funnel và seller/shop aggregate, không là tài chính thật |
| Platform | OpenAPI, migration, observability, CI/CD, backup/restore, security, quality gates |

## 6. Phạm vi mở rộng sau MVP

Flash sale, recommendation cá nhân hóa, advertising/sponsored placement, outbound affiliate, seller analytics nâng cao, bulk import và automation; mỗi năng lực có đặc tả trong [domains](../domains/README.md) và chỉ bật theo [roadmap](../delivery/roadmap.md). Quảng cáo thanh toán thật và hoa hồng đối tác thật là **hướng kinh doanh đã xác nhận**, nhưng mô hình giá, provider, đối tác, thời điểm và điều kiện bật vẫn là quyết định mở; cần đặc tả/contract/pháp lý/thuế/an toàn/ledger-đối soát riêng. Kết nối **đơn mua Cartasy** với tiền thật hoặc giao hàng thật là thay đổi sản phẩm khác và chưa được chấp thuận. Không bật bất cứ luồng tiền thật nào chỉ bằng biến môi trường.

## 7. Quy tắc chung và chấp nhận MVP

- Tiền VND là integer minor unit server-owned, API serialize dạng chuỗi; không float. Client không quyết định giá, stock, discount, actor, shop, payment/order state.
- Idempotency cho checkout, mock settlement, refund, invitation, notification fanout; outbox/event at-least-once, consumer dedupe. Giá/tồn kho và quyền kiểm primary ở thời điểm hành động.
- Buyer không thấy PII seller không cần; seller không xem buyer/shop khác. Mọi thao tác nhạy cảm có reason/audit và MFA/recent auth khi policy yêu cầu.
- Multi-shop E2E: hai shop, hai seller, hai buyer; checkout ra đúng shop orders, seller chỉ xử lý của mình, notification đúng người, refund không vượt paid simulated.
- Các kịch bản mock success/failure/timeout, stale quote, out-of-stock, seller bị suspend, retry sau timeout, return/support và announcement đều có UI, API và test.
- Mobile, accessibility WCAG 2.2 AA cho luồng chính, SEO cho public catalog/shop, privacy/consent, backup/restore và monitoring qua release gate. Dữ liệu seed/demo và chỉ số không được ngụy trang thành giao dịch hoặc review thật.

## 8. Giả định kỹ thuật và điểm cần xác minh

Giả định MVP dùng email làm định danh, Việt Nam/VND là thị trường đầu, giao diện Web `vi`/`en`, một kho ảo mỗi listing, một voucher code/order, support dạng ticket bất đồng bộ và Web Push chỉ khi người dùng opt-in. Các giả định này có owner và điều kiện xác minh ở [Audit & open decisions](../delivery/audit-and-open-decisions.md); thay đổi không được âm thầm sửa domain contract.
