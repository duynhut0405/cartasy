---
doc_id: UX-DESIGN
title: Cartasy UI Design System
status: accepted
owner: Cartasy Design & Product
last_updated: 2026-10-20
version: 4.4.1
source_of_truth: Token, component, nội dung, responsive và accessibility
review_cycle: Khi thay đổi brand hoặc interaction pattern
---

# Hệ thống giao diện

## 1. Nguyên tắc

Cartasy phải có nhịp và chức năng của marketplace thật: catalog rõ, giá VND dễ đọc, shop/variant minh bạch, giỏ và order có trạng thái đầy đủ. Trải nghiệm mô phỏng được trình bày thành thật, không làm người dùng tưởng đã trả tiền hoặc sắp nhận hàng. UI không dùng “credit”, “ví điểm”, “nhận ngân sách” làm phương tiện thanh toán mock order. Outbound affiliate **sau MVP** là CTA riêng, công bố rõ mua thật ở đối tác; không thay nút checkout Cartasy một cách âm thầm. [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

## 2. Token và responsive baseline

Tài liệu này sở hữu semantic token; `apps/web` và `apps/admin` hiện thực các CSS variables cần dùng trong `src/app/globals.css` của từng app. Tailwind 4 map vào token, không hardcode màu khắp feature. Baseline trung tính có thể đổi brand mà không đổi component API:

| Token | Giá trị baseline | Dùng cho |
|---|---|---|
| `surface.canvas` | `#f8fafc` | Nền trang |
| `surface.card` | `#ffffff` | Card/dialog |
| `text.primary` | `#111827` | Nội dung |
| `text.muted` | `#475569` | Meta |
| `border.default` | `#cbd5e1` | Viền |
| `action.primary` | `#0f766e` | Hành động chính |
| `status.danger` | `#b91c1c` | Lỗi |
| `status.warning` | `#92400e` | Cảnh báo |
| `status.success` | `#047857` | Thành công |
| `simulation.surface` | `#fff7ed` | Thông báo mô phỏng |
| `simulation.text` | `#9a3412` | Chữ mô phỏng |

Spacing 4/8/12/16/24/32/48/64 px; radius 8/12/16 px; touch target tối thiểu 44×44 px. Typography dùng system sans, scale có phân cấp H1/H2/H3/body/meta; giá/tổng dùng tabular numerals. Mobile trước, grid sản phẩm 2 cột từ điện thoại phù hợp, 3–5 cột desktop; chi tiết/giỏ/checkout một cột mobile, hai cột khi đủ rộng. Không dựa vào màu để truyền trạng thái.

### Tổ chức file thiết kế và ownership

Web/Admin dùng component shadcn/ui bản **Radix** trực tiếp trong `src/shared/ui` của từng app. Add component theo nhu cầu màn hình và review source được sinh; nếu thiếu element hoặc hành vi, compose từ shadcn/Radix và semantic HTML trong app. Domain visual ở `entities`, thao tác ở `features`, section ở `widgets`, route ở `app`; không tạo một UI kit riêng cho từng nghiệp vụ. Token có nhóm màu semantic, typography, spacing, radius, shadow, z-index, breakpoint/container và motion duration/easing; tên token mô tả vai trò thay vì màu cụ thể. Khi đổi token dùng chung, cập nhật cả Web/Admin và kiểm contrast/visual. App/page/feature dùng Tailwind utilities và token, không viết CSS thuần hoặc CSS Modules; CSS global chỉ cho Tailwind import, variables/theme và foundation thật sự cần thiết. [ADR-0014](../decisions/ADR-0014-app-owned-shadcn-components.md).

Mỗi component có contract: props/slots, size/variant, trạng thái default/hover/focus/disabled/loading/error/empty, responsive, keyboard/screen reader, và ví dụ text `vi`/`en`. Form primitive không tự xử lý API error; feature map lỗi theo field và summary. Layout template chuẩn: `StoreShell`, `AccountShell`, `AuthShell`, `SellerShell`, `PlatformShell`; shell cung cấp skip link, header/nav/breadcrumb/main/footer, không chứa business rule. Trang chỉ compose section và data; một section có spacing/container thống nhất qua recipe, không hardcode riêng mỗi page.

Ảnh có placement contract (hero, card, gallery, avatar), aspect ratio, crop focal point, responsive `sizes`, skeleton/error fallback và alt text theo locale đã duyệt. Icon có accessible name khi mang nghĩa, decorative icon ẩn với assistive tech. Motion dùng cho phản hồi UI, tôn trọng reduced motion. Bảng Admin ở mobile có pattern xem chi tiết/overflow rõ, không ép bảng nhiều cột gây mất hành động.

## 3. Component chuẩn

Các element cần dùng: `Button`, `Link`, `TextField`, `Select`, `Checkbox`, `RadioGroup`, `Dialog`, `AlertDialog`, `Tabs`, `Badge`, `Table`, `Pagination`, `Skeleton`, `Toast`, `InlineAlert`, `EmptyState` và `ErrorState`. Add component shadcn tương ứng khi có; các pattern còn thiếu được compose trong app khi màn hình cần. Component có size, variant, disabled/loading/error/focus state và keyboard contract.

Domain component thuộc feature/app:

| Feature | Component |
|---|---|
| Catalog | `ProductCard`, `ProductGallery`, `VariantPicker`, `ListingSelector`, `ShopBadge`, `PriceBlock`, `StockStatus` |
| Search | `SearchBox`, `FilterPanel`, `SortControl`, `AppliedFilters` |
| Cart | `ShopCartGroup`, `CartItemRow`, `CartSummary`, `MergeWarnings` |
| Checkout | `AddressSelector`, `CouponEntry`, `OrderPriceBreakdown`, `SimulationNotice`, `PaymentMethodPicker`, `CheckoutConfirmation` |
| Orders | `OrderStatusBadge`, `ShopOrderTimeline`, `PaymentStatusPanel`, `CancellationDialog` |
| Notifications | `NotificationList`, `UnreadBadge` |
| Wishlist/review/return/support | `WishlistButton`, `ReviewForm`, `ReturnRequestForm`, `TicketThread` |
| Admin | `PublishChecklist`, `MoneyInput`, `InventoryAdjustmentDialog`, `OrderTransitionPanel`, `AuditTrail` |

Mọi component có test/story hoặc ví dụ trạng thái cần thiết. Form không dùng toast làm kênh duy nhất cho lỗi; lỗi gắn với field và summary.

## 4. Trang F1

- Trang chủ: điều hướng danh mục, tìm kiếm, module sản phẩm/shop đã duyệt; không tạo lượt bán/review giả.
- Search/category: query/filter/sort trong URL, skeleton, empty state có cách đổi filter; cursor giữ thứ tự ổn định.
- Product Detail: tên/ảnh/variant, listing theo shop, giá VND, tình trạng hàng, thông tin shop, chính sách mô phỏng và CTA thêm giỏ. Nếu nhiều shop, mỗi giá gắn đúng shop.
- Shop: danh tính, mô tả, listing public. Không dùng nhãn “đã xác minh” nếu chưa có trạng thái verification thật.
- Cart: nhóm shop, quantity, item unavailable, subtotal/phí dự kiến và link checkout.
- Checkout: địa chỉ mô phỏng, quote từng shop, coupon, phí, tổng, mock method, xác nhận mô phỏng rõ cạnh nút. Tổng thay đổi yêu cầu người dùng xác nhận lại.
- Payment/result: pending/success/failure/timeout tách rõ, không hiển thị “đã trừ tiền”; cung cấp mã đơn và đường xem đơn.
- Orders/detail: parent order, shop order, payment attempt và timeline giao nhận có nhãn mô phỏng; nút hủy theo state.
- Wishlist: guest/member sync, unavailable item và chọn listing trước khi thêm giỏ.
- Reviews: nhãn “trải nghiệm mô phỏng”, moderation state, không gắn badge mua/nhận hàng thật.
- Returns/refunds: chọn item/quantity, thời hạn, timeline quyết định và nhãn hoàn tiền mô phỏng.
- Support: ticket async, reply/attachment, trạng thái/SLA, không hứa live chat.
- Notification center: order transactional, product/promo/system tabs, unread/read/archive, opt-in email/push và trạng thái permission denied; push không được hỏi khi page load.
- Khu Seller trong Admin workspace MVP: onboarding, shop switcher, listing/stock/order/promotion/support/report có scope và suspended state; khu Platform có navigation/capability riêng trong cùng app.
- Admin CMS/campaign: draft/preview/review/schedule/publish/rollback, second approver critical, delivery progress và pause.
- Account/auth: quyền riêng tư và lỗi session có hướng xử lý.

## 5. Nội dung bắt buộc

Mọi UI copy thuộc namespace bản dịch Web `vi`/`en`; content catalog/shop/CMS thuộc owner domain và có trạng thái publish riêng. Nội dung thông báo/email chọn template theo locale người nhận. QA kiểm câu tiếng Anh dài, plural/placeholder, wrapping, focus order và disclosure mô phỏng ở cả hai ngôn ngữ. Locale switcher hiển thị tên ngôn ngữ tự thân (“Tiếng Việt”, “English”), nêu rõ khi trang đích chưa có bản dịch. [ADR-0012](../decisions/ADR-0012-multilingual-storefront.md).

Tại checkout và nút đặt hàng:

> Giao dịch mô phỏng: Cartasy không thu tiền và không giao hàng thật.

Tại kết quả thanh toán:

> Thanh toán mô phỏng thành công. Không có tiền được chuyển.

Tại trạng thái giao hàng:

> Tiến trình giao nhận này là mô phỏng; sẽ không có bưu kiện được gửi.

Email/notification mở bằng hoặc gần CTA có nhãn “Đơn hàng mô phỏng”. Không dùng logo ngân hàng/ví hoặc tên hãng vận chuyển thật như đang được tích hợp. Địa chỉ được giải thích là thông tin nhập để thử checkout; người dùng không cần nhập địa chỉ thật. Không gọi outcome mock là “biên lai” hoặc “hóa đơn”.

## 6. Admin/seller

Admin layout có navigation theo context Catalog, Listing & Inventory, Promotions, Orders, Returns, Support, Shops, Moderation, Operations. Route chỉ hiển thị khả năng phù hợp role, nhưng API vẫn quyết định quyền. Bảng có filter, cursor, empty/error state; mutation hiển thị version conflict và audit reason. Seller F1 luôn có shop switcher rõ ràng, dữ liệu shop cũ bị clear khi đổi; trạng thái draft/review/published/suspended tách biệt.

## 7. Accessibility và SEO

Mục tiêu WCAG 2.2 AA cho luồng chính: keyboard hoàn chỉnh, focus visible, skip link, semantic heading, label/error relation, dialog focus trap/return, aria-live có kiểm soát cho payment status, alt text sản phẩm và reduced motion. Kiểm tra contrast tự động và thủ công trên token. Không dùng countdown/FOMO giả. Product/category/shop public có canonical metadata và structured data chỉ cho thông tin thật của mô phỏng, không xuất `Offer`/rating schema gây hiểu nhầm về giao dịch thật. Auth/cart/checkout/order/account đặt `noindex`.

## 8. Sponsored và outbound affiliate sau MVP

Sponsored Product/Shop/Search/Display luôn có nhãn “Quảng cáo” ngay tại card/banner và giữ rõ nguồn shop/giá; không giả là kết quả hữu cơ hoặc quyền xác minh shop. External affiliate offer có nhãn “Liên kết tiếp thị”, tên đối tác, CTA “Xem/mua thật tại [đối tác]” và thông báo trước redirect: “Bạn sẽ rời Cartasy. Đơn hàng, thanh toán, giao hàng và hoàn tiền trên đối tác là giao dịch thật do đối tác xử lý.” Nút này không dùng wording “Đặt đơn Cartasy”, không thêm vào giỏ hoặc áp voucher mô phỏng. Giá/stock đối tác chỉ hiện khi có quyền sử dụng, nguồn và thời điểm cập nhật; khi không chắc, chỉ dẫn đến đối tác xem giá hiện tại. Sponsored/affiliate disclosure phải đọc được trên mobile, screen reader và ở deep link; không ẩn trong Terms. [Advertising](../domains/advertising.md), [Affiliate](../domains/affiliate.md).
