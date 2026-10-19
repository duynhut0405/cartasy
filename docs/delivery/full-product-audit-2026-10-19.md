---
doc_id: DELIVERY-FULL-PRODUCT-AUDIT-20261019
title: Cartasy — Full Product, Feature and Monetization Audit
status: audit
owner: Product & Architecture
last_updated: 2026-10-19
version: 1.1.0
source_of_truth: Bằng chứng audit và gap của bộ tài liệu hiện hành; không thay domain owner
---

# Cartasy — Full Product & Feature Audit

## 1. Kết luận điều hành và phương pháp

Đã đối chiếu toàn bộ tài liệu hiện hành trong `docs/` theo scope → domain → data/API → app/worker → test/operations. Workspace chưa có `apps/`, `packages/`, manifest, OpenAPI, migration, CI hay source; vì vậy đây là **kiểm thử kịch bản trên tài liệu**, không phải E2E ứng dụng. [Index § Cây tài liệu](../README.md), [Checklist § F0–F3](build-checklist.md), [Testing § Bằng chứng](../platform/testing-quality.md). Kiến trúc F1 có một Admin workspace cho Seller/Platform theo [ADR-0011](../decisions/ADR-0011-unified-admin-seller-workspace.md); ranh giới monetization theo [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

**Kết luận:** Blueprint F1 đa seller/mock commerce khá rộng và có ranh giới module tốt. Luồng Buyer miễn phí, đơn Cartasy mô phỏng, quảng cáo có thể thu tiền thật và affiliate outbound mua thật trên đối tác nay đã tách rõ. **Chưa thể gọi toàn bộ MVP “sẵn sàng code trọn gói”**: quy tắc parent order nhiều shop, shipment failure/actor matrix, promotion allocation, một số audience/role/UX và Product decisions còn mở; không có artifact hay gate đã chạy. F3 có boundary để mở rộng nhưng **ad billing, partner contract/API và revenue recognition chưa sẵn sàng triển khai**. [Orders § trạng thái](../domains/orders.md), [Shipping § Quy trình](../domains/shipping-simulation.md), [Ad billing § Release gate](../domains/advertising-billing.md), [Affiliate § Partner due diligence](../domains/affiliate.md).

Ký hiệu: **Confirmed** = yêu cầu mới hoặc scope đã được duyệt; **Assumed** = baseline kỹ thuật cần xác minh; **Proposed** = audit đề xuất, chưa thành cam kết; **Open** = cần Product/Finance/Legal/Ops quyết định. Độ sâu **2** = rule/state/owner/API-data map có nhưng còn test oracle/DTO; **1** = định hướng; **0** = chưa có đặc tả; **3** = contract/example/state/error/test đủ và được review, hiện chưa có module nào đạt 3. Trạng thái **Docs** không là trạng thái **Implementation**. P0 = MVP bắt buộc; P1 = giai đoạn tăng trưởng kế tiếp hoặc F1 gate đã xác nhận cần chia milestone; P2 = monetization/mở rộng dài hạn. [Scope § 1–8](../system/product-scope.md), [Roadmap § F0–F3](roadmap.md).

## 2. Feature Coverage Matrix

Mỗi hàng là feature/subfeature có actor, phase, trạng thái tài liệu và đường dẫn section. `Có/2` không có nghĩa code đã tồn tại. Phase của self-registration đang [M-01 Open](audit-and-open-decisions.md); hai seller hoạt động F1 vẫn Confirmed. Các mục Proposed không tự thêm vào MVP.

### Buyer, catalog và khám phá

| Feature/subfeature | Actor | Loại/ưu tiên | Docs | Bằng chứng section/gap |
|---|---|---|---|---|
| Home, collection, category/shop landing | Guest/Buyer/Admin | Confirmed P0/F1 | Có/2 | [CMS § Block type/Contract](../domains/cms-content.md), [Flows § FLOW-01](../system/business-flows.md); collection fallback có, layout content limits thiếu DTO. |
| Canonical product/variant/SKU/attribute/media | Admin/Seller/Buyer | Confirmed P0/F1 | Có/2 | [Catalog § Mô hình/Validation](../domains/catalog.md), [Data model § Catalog](../system/data-model.md); taxonomy/variant constraint nền có. |
| Listing theo shop, giá VND/tồn kho ảo | Seller/Buyer/Admin | Confirmed P0/F1 | Có/2 | [Catalog § Contract](../domains/catalog.md), [Inventory § Adjustment](../domains/inventory.md); public hint không là stock authoritative. |
| Product detail/chọn variant và shop | Guest/Buyer | Confirmed P0/F1 | Có/2 | [Flows § FLOW-01](../system/business-flows.md), [Design § Trang F1](../system/design-system.md); UX khi variant không có listing chưa có decision table. |
| Full-text/typo/prefix/Unicode search | Guest/Buyer | Confirmed P0/F1 | Có/2 | [Discovery § Search/Contract](../domains/discovery.md); tìm không dấu cần fixture/acceptance riêng. |
| Filter/sort/facet/cursor/shop filter | Guest/Buyer | Confirmed P0/F1 | Có/2 | [Discovery § Contract filter](../domains/discovery.md), [API § Catalog](../system/api-contracts.md); `popular` chưa định nghĩa nguồn/độ tin cậy. |
| Autocomplete/keyword suggestion | Guest/Buyer | Proposed P1/F2 | Thiếu/0 | [Discovery § Search](../domains/discovery.md) chỉ typo/prefix result, chưa suggestion endpoint/ranking/abuse limit. |
| Search history | Buyer | Proposed P1/F2 | Thiếu/0 | [Analytics § event dictionary](../system/analytics.md) không phải lịch sử tìm kiếm; cần consent/delete/retention. |
| Recently viewed | Guest/Buyer | Proposed P1/F2 | Thiếu/0 | [Feature map § Buyer/discovery](../system/feature-map.md) nêu khả năng sau MVP, chưa có data/API/UI/consent. |
| Wishlist guest/server/merge | Guest/Buyer | Confirmed P1/F1 | Có/2 | [Wishlist § Contract](../domains/wishlist.md), [Flows § FLOW-06](../system/business-flows.md). |
| Follow shop/category | Buyer | Proposed P1/F2 | Thiếu/0 | [Wishlist § Contract](../domains/wishlist.md) để sau MVP; notification “follower” chưa thể là F1. |
| Recommendation rule-based/personalized | Buyer | Confirmed F2/F3 P1/P2 | Định hướng/1 | [Recommendations § toàn bộ](../domains/recommendations.md): eligibility/fallback có, model/API/experiment chưa đủ. |
| Gamification/reward ảo an toàn | Buyer | Proposed P1/F2 | Thiếu/0 | [Design § Accessibility](../system/design-system.md) cấm FOMO giả; reward không thể dùng làm tiền/credit. |

### Cart, transaction, after-sales

| Feature/subfeature | Actor | Loại/ưu tiên | Docs | Bằng chứng section/gap |
|---|---|---|---|---|
| Guest cart/login merge/server cart | Guest/Buyer | Confirmed P0/F1 | Có/2 | [Cart § Cart](../domains/cart-checkout.md), [Flows § FLOW-02](../system/business-flows.md); retry/key rõ. |
| Voucher shop/toàn sàn/usage quota | Buyer/Seller/Admin | Confirmed P0/F1 | Có/2 | [Promotions § Priority](../domains/promotions-vouchers.md); stacking/return coupon là Assumed A-03. |
| Quote nhiều shop, shipping method từng shop | Buyer | Confirmed P0/F1 | Có/2 | [Cart § Quote](../domains/cart-checkout.md), [API § 5–6](../system/api-contracts.md), [Data model § Order](../system/data-model.md); contract vừa bổ sung, chưa có OpenAPI/test. |
| Checkout atomic/idempotent/split order | Buyer/API/Worker | Confirmed P0/F1 | Có/2 | [Data model § Invariant](../system/data-model.md), [Architecture § Checkout](../system/architecture.md). |
| Mock payment success/failure/timeout | Buyer/Worker | Confirmed P0/F1 | Có/2 | [Payment § State/Attempt](../domains/payment-simulation.md): không nhận outcome từ browser. |
| Order list/detail/reorder sau failure | Buyer | Confirmed P0/F1 | Có/2 | [API § 6](../system/api-contracts.md), [Flows § FLOW-03/04](../system/business-flows.md); reorder là cart/checkout mới, chưa có one-click endpoint. |
| Parent/shop order state/cancel từng shop | Buyer/Seller/Admin | Confirmed P0/F1 | Một phần/2 | [Orders § Permission matrix](../domains/orders.md); parent aggregation với child hỗn hợp chưa khép. |
| Shipping simulated/ETA/timeline | Buyer/Seller/Admin/Worker | Confirmed P0/F1 | Một phần/2 | [Shipping § Quy trình](../domains/shipping-simulation.md); shipment↔shop-order mapping/actor chưa khép. |
| Giao thất bại/retry/điều chỉnh ETA | Buyer/Seller/Admin/Worker | Proposed P1/F1? | Thiếu/0 | [Shipping § Quy trình](../domains/shipping-simulation.md) chỉ ready→delivered/cancel; quyết định branch/compensation còn Open. |
| Return/refund partial/full mô phỏng | Buyer/Seller/Admin | Confirmed P1/F1 | Có/2 | [Returns § State](../domains/returns-refunds.md): cap/quantity có; cửa sổ/ship fee allocation Assumed. |
| Review/rating trải nghiệm mô phỏng | Buyer/Seller/Admin | Confirmed P1/F1 | Có/2 | [Reviews § Luồng](../domains/reviews.md): không chứng nhận dùng hàng thật. |
| Support ticket/report/appeal | Buyer/Seller/Admin | Confirmed P1/F1 | Có/2 | [Support § Dữ liệu](../domains/customer-support.md), [Risk § State](../domains/risk-moderation.md); appeal API còn gián tiếp. |
| Flash sale quota/atomic stock | Buyer/Seller/Admin | Confirmed P1/F2 | Định hướng/1 | [Flash sale § toàn bộ](../domains/flash-sale.md); chưa có API DTO/full UX. |

### Notification, engagement, Seller, Admin và platform

| Feature/subfeature | Actor | Loại/ưu tiên | Docs | Bằng chứng section/gap |
|---|---|---|---|---|
| Inbox read/unread/archive | Buyer/Seller/Admin | Confirmed P1/F1 | Có/2 | [Communications § 5](../domains/communications.md), [API § 8](../system/api-contracts.md), [Data model § Platform](../system/data-model.md). |
| Transactional email/Web Push opt-in/SSE | Buyer/Seller/Admin/Worker | Confirmed P1/F1 | Có/2 | [Communications § 2–7](../domains/communications.md); external delivery không hứa exactly-once. |
| Product-new/promo/system campaign | Seller/Admin/Buyer | Confirmed P1/F1 | Có/2 | [Communications § 4](../domains/communications.md): F1 audience opt-in chung, follow audience F2 nếu triển khai. |
| Notification event→recipient→channel→deep link matrix | Buyer/Seller/Admin/Worker | Confirmed P1/F1 | Có/2 | [Communications § 3 — ma trận trigger](../domains/communications.md) đã có event logic, recipient, kênh, template, deep link và offline; template DTO/severity còn cần chốt theo slice. |
| Shop onboarding/review/activation | Applicant/Admin | Confirmed F1 baseline, phase Open | Có/2 | [Marketplace § 2](../domains/marketplace-seller.md), [API § 10/12](../system/api-contracts.md); M-01 chưa chốt. |
| Membership/invite/owner transfer/MFA | Seller/Admin | Confirmed P0/F1 | Có/2 | [Marketplace § 3](../domains/marketplace-seller.md), [Data model § Identity](../system/data-model.md): contract cơ bản vừa thêm, chưa test oracle. |
| Product proposal/matching/listing publish | Seller/Admin | Confirmed P0/F1 | Có/2 | [Catalog § Contract/ProductProposal](../domains/catalog.md), [API § 12](../system/api-contracts.md); chưa có example DTO/decision reason matrix. |
| Seller stock/order/promo/support/report | Seller | Confirmed P0–P1/F1 | Có/2 | [Admin workspace § Seller](../apps/admin/README.md), [Marketplace § 4](../domains/marketplace-seller.md). |
| Admin user/role/suspend/access audit | Admin | Confirmed P0/F1 | Có/2 | [API § 9](../system/api-contracts.md), [Identity § Admin operation](../domains/identity-profile.md); route cơ bản có, grant workflow chưa fully specified. |
| Admin CMS/moderation/campaign/report/operations | Admin | Confirmed P1/F1 | Có/2 | [Admin guide § Routing](../apps/admin/README.md), [CMS § Contract](../domains/cms-content.md), [Risk § API](../domains/risk-moderation.md). |
| Auth/tenant/PII/consent/CSRF/MFA | All | Confirmed P0/F0–F1 | Có/2 | [Security § 1–9](../system/security-privacy.md); provider snapshot/HTTPS gate chưa chạy. |
| OpenAPI/schema/migrations/outbox/jobs | API/Worker | Confirmed P0/F0 | Blueprint/1 | [API § 11](../system/api-contracts.md), [Migrations § Workflow](../platform/database-migrations.md), [Worker § Poll](../apps/worker/README.md); không có artifact. |
| Caching/search/SEO/observability/backup | Platform | Confirmed P0/F0–F1 | Blueprint/1 | [Architecture § 5–8](../system/architecture.md), [Operations § 3–8](../platform/deployment-operations.md); chưa có provider/drill. |

### Monetization sau MVP

| Feature/subfeature | Actor | Loại/ưu tiên | Docs | Bằng chứng section/gap |
|---|---|---|---|---|
| Sponsored Product/Shop/Search/Display/Banner | Advertiser/Seller/Admin/Buyer | Confirmed hướng P2/F3 | Boundary/1 | [Advertising § Sản phẩm/Placement](../domains/advertising.md); slot inventory/format/pricing chưa duyệt. |
| Advertiser self-service/creative approval | Advertiser/Seller/Admin | Confirmed hướng P2/F3 | Boundary/1 | [Advertising § Placement/Contract](../domains/advertising.md); account/role/approval DTO chưa chốt. |
| CPC/CPM/fixed, budget/cap/fraud | Advertiser/Finance/Admin | Open P2/F3 | Đề xuất/1 | [Advertising § Targeting](../domains/advertising.md), [Open M-03](audit-and-open-decisions.md); không chọn pricing mặc định. |
| Ad payment/invoice/refund/ledger/revenue | Advertiser/Finance | Confirmed hướng P2/F3, điều kiện mở | Boundary/1 | [Ad billing § Aggregate/Gate](../domains/advertising-billing.md); pháp nhân/provider/tax/accounting chưa duyệt. |
| External offer/partner content/link | Partner/Admin/Buyer | Confirmed hướng P2/F3 | Boundary/1 | [Affiliate § Hai hành trình/Quyền](../domains/affiliate.md); rights/API chưa có hợp đồng. |
| Outbound CTA/redirect/consent | Buyer/Admin | Confirmed hướng P2/F3 | Boundary/2 | [Affiliate § Luồng](../domains/affiliate.md), [Design § Sponsored/outbound](../system/design-system.md); consent-off policy Open. |
| Partner conversion/commission/reversal | Partner/Finance/Admin | Confirmed hướng P2/F3 | Boundary/1 | [Affiliate § Luồng/Partner due diligence](../domains/affiliate.md); report schema/settlement tùy partner. |
| Real revenue dashboard tách mock value | Finance/Admin/Seller | Confirmed hướng P2/F3 | Boundary/2 | [Reporting § Ranh giới monetization](../domains/reporting.md), [Analytics § KPI](../system/analytics.md); recognition policy Open. |

## 3. Findings: Missing, Incorrect, Overengineering

Các mã `C/H/M/L` trong §3 là **finding của báo cáo**; mã `M-*` tại [Open Decisions](audit-and-open-decisions.md) là **quyết định Product/Finance/Legal** và có thể trùng phần số. Khi dẫn lại phải ghi “finding M-02” hoặc “Open Decision M-02”.

| ID/mức/trạng thái | Bằng chứng cụ thể | Tác động và xử lý |
|---|---|---|
| **C-01 Critical — đã sửa trong docs** | [Affiliate](../domains/affiliate.md) và [Advertising billing](../domains/advertising-billing.md) cần chứng cứ doanh thu riêng; mock order không chứng minh tiền thật. | [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md), [Scope § 1, 3, 6](../system/product-scope.md), domain/analytics/reporting tách ba luồng. Chưa bật tiền thật. |
| **C-02 Critical — chưa triển khai** | [Checklist § F0/F1](build-checklist.md) toàn `[ ]`; [Local § bootstrap](../platform/local-development.md) nói manifest/Compose là blueprint. | Không có source/migration/OpenAPI/gate; chỉ đủ bắt đầu scaffold có kiểm soát, không thể tuyên bố MVP sẵn sàng release. |
| **H-01 High — mở** | [Orders § trạng thái/Permission](../domains/orders.md), [Data model § Order](../system/data-model.md) chưa có hàm dẫn xuất parent khi child cancelled/returned/delivered hỗn hợp. | API/UI/notification/report có thể khác nhau. Cần bảng truth table parent×child×payment/refund, lock order và test tất cả tổ hợp trước code Orders F1. |
| **H-02 High — mở** | [Shipping § Quy trình](../domains/shipping-simulation.md) dùng shipment `ready→packed`, [Orders § Permission](../domains/orders.md) dùng shop order `confirmed→processing→packed`; worker và seller cùng chuyển. Giao thất bại/tracking history chưa có state. | Chốt actor/state mapping và failure/retry/compensation/ETA policy; không cho auto worker skip seller command. |
| **H-03 High — mở** | [Roadmap § F1/F2](roadmap.md) và [Open M-01](audit-and-open-decisions.md): baseline self-registration F1 nhưng yêu cầu mới cho phép để sau. | Product chốt phase; nếu hoãn, bổ sung Admin provisioning flow hai seller F1 và sửa scope/API/app/gate cùng lúc. |
| **H-04 High — mở** | [Communications § Campaign](../domains/communications.md) F1 chỉ có opt-in chung; [Wishlist § Contract](../domains/wishlist.md) để follow sau MVP. | Kịch bản “shop mới báo follower” chưa khả thi F1. Chốt follow phase hoặc đánh dấu scenario future; không suy follower từ wishlist/order. |
| **H-05 High — mở cho F3** | [Ad billing § điều kiện bật/Gate](../domains/advertising-billing.md), [Open M-03](audit-and-open-decisions.md) chưa có legal entity, pricing, provider, tax/invoice, revenue recognition. | Không thể triển khai billing thật từ domain sketch; cần commercial/Finance decision và contract trước schema/API/code. |
| **H-06 High — mở cho F3** | [Affiliate § Partner due diligence](../domains/affiliate.md) có yêu cầu xác minh, chưa có Cartasy account/contract/API permission/report schema với partner. | Không thể gọi click là hoa hồng hoặc hiển thị giá/ảnh không có quyền; due diligence theo quốc gia/chương trình và report reconciliation. |
| **H-07 High — đã sửa contract, chưa kiểm thử** | [Cart § Quote](../domains/cart-checkout.md) trước đây yêu cầu method từng shop nhưng [API § 5–6](../system/api-contracts.md) thiếu selection. | Đã thêm `shippingSelections[]` vào quote/checkout và snapshot; cần OpenAPI, migration và race test khi triển khai. |
| **H-08 High — đã sửa contract, chưa kiểm thử** | [Marketplace § 2–3](../domains/marketplace-seller.md) có activation/invite; [API § 10/12](../system/api-contracts.md) trước đây thiếu accept/activate/owner transfer. | Đã thêm endpoint/owner table tối thiểu; cần DTO, auth/replay và E2E. |
| **M-01 Medium — mở** | [Discovery § Search](../domains/discovery.md) có full-text/typo nhưng không có autocomplete/history; `popular` không có nguồn/ranking version. | Đề xuất F2 autocomplete/history/recently viewed; định nghĩa `popular` hoặc bỏ sort cho tới khi đủ dữ liệu sạch. |
| **M-02 Medium — mở** | [Promotions § Priority](../domains/promotions-vouchers.md), [Returns § State](../domains/returns-refunds.md) dùng giả định stacking/coupon restore/ship fee refund. | Chốt allocation và rounding snapshot cho cancel từng shop/return partial; test concurrent quota/retry. |
| **M-03 Medium — đã bổ sung ma trận, còn template/gate** | [Communications § 3 — ma trận trigger](../domains/communications.md) hiện truy vết từng event F1 tới recipient, channel, template, link, offline/fallback; campaign follow còn thiếu data/API. | Chốt template DTO, severity và policy marketing trong từng slice, test wrong-shop/revoke/duplicate/consent trước release. |
| **M-04 Medium — đã sửa wording** | [Orders § chi tiết](../domains/orders.md) từng gọi “invoice mô phỏng” trong khi [Design § Nội dung](../system/design-system.md) cấm “hóa đơn” cho mock outcome. | Đã đổi thành “xác nhận đơn mô phỏng”; invoice thật chỉ thuộc ad billing sau này. |
| **M-05 Medium — đã sửa kỳ vọng** | [Architecture § Event](../system/architecture.md) dùng at-least-once; [Communications § Transactional](../domains/communications.md) trước đây dễ hiểu external send exactly-once. | Docs nay phân biệt unique inbox/delivery intent với SMTP/Web Push có thể duplicate sau crash; test/runbook cần phản ánh. |
| **M-06 Medium — mở** | [CMS § Contract](../domains/cms-content.md) còn “maximum payload chốt trong DTO”; [API § 9/12](../system/api-contracts.md) nhiều route dạng inventory, chưa có response/error/example chi tiết. | Agent chỉ nên code vertical slice sau khi viết DTO/OpenAPI/test oracle tương ứng, không coi tên route là contract hoàn tất. |
| **L-01 Low — đã sửa** | [Discovery § Search](../domains/discovery.md) từng ghi shop filter F2 dù [API § Catalog](../system/api-contracts.md) đã có F1. | Đã thống nhất shop filter F1. |

**Overengineering cần tránh:** không thêm Redis/BullMQ/FCM/WebSocket, native app, ad auction, payout, generic wallet hoặc real buyer payment vào F1 khi chưa có use case/gate; [Stack § Thư viện](../platform/stack-dependencies.md), [Integrations § Realtime](../platform/external-integrations.md), [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md). Ad billing và affiliate outbound được giữ ở boundary/contract tương lai, **không** tạo bảng/route F3 rỗng tại F1. Follow/reward/autocomplete là đề xuất, không tự lấy tính năng đối thủ làm yêu cầu.

## 4. Cross-module audit và kịch bản E2E trên tài liệu

Trạng thái **Đủ nền** = có tuyến owner/rule/data/API/event nhưng chưa có code/test; **Một phần** = thiếu quyết định/contract; **Chưa** = chủ yếu định hướng. Cột event/noti chỉ gọi event đã có trong docs hoặc ghi “dự kiến”, không coi đó là event schema đã triển khai. Mọi kịch bản phải được thực hiện với **2 shop, 2 seller, 2 buyer** và actor/tenant guard. [Testing § Ma trận luồng](../platform/testing-quality.md).

| #/kịch bản | Rule/UI | Data/API | Event/notification và edge case | Phủ |
|---|---|---|---|---|
| 1. Tìm/lọc/xem/thêm giỏ | [Discovery § Contract](../domains/discovery.md), [Flows § FLOW-01](../system/business-flows.md) | [Data § Catalog/Cart](../system/data-model.md), [API § 4–5](../system/api-contracts.md) | [Catalog § event](../domains/catalog.md); unpublished, no-diacritic, stale stock, cursor tamper | Đủ nền |
| 2. Mixed-shop quote/voucher/checkout | [Cart § Quote/Place](../domains/cart-checkout.md), [Promotions § Priority](../domains/promotions-vouchers.md) | [Data § Invariant](../system/data-model.md), [API § 5–6](../system/api-contracts.md) | [Architecture § Checkout/outbox](../system/architecture.md); last stock, shop suspend, price change, double submit, timeout after commit | Một phần: allocation policy |
| 3. Seller nhận/xử lý, auto ship | [Orders § Permission](../domains/orders.md), [Shipping § Quy trình](../domains/shipping-simulation.md) | [Data § Order/fulfillment](../system/data-model.md), [API § 10](../system/api-contracts.md) | [Worker § scheduler](../apps/worker/README.md), [Communications § Taxonomy](../domains/communications.md); seller/worker race, suspend after paid | Một phần: H-01/H-02 |
| 4. Tracking history và notification | [Shipping § Timeline](../domains/shipping-simulation.md), [Flows § FLOW-04/08](../system/business-flows.md) | [Data § shipment_events/notifications](../system/data-model.md), [API § 6/8](../system/api-contracts.md) | [Communications § 3 — ma trận trigger](../domains/communications.md); duplicate event, wrong buyer/shop, out-of-order, delayed projection | Đủ nền; shipping actor/state H-02 còn mở |
| 5. Tab đóng, Web Push opt-in | [Communications § Web Push](../domains/communications.md), [Web § API/UI](../apps/web/README.md) | [Data § push_subscriptions](../system/data-model.md), [API § 8](../system/api-contracts.md) | [Worker § handlers](../apps/worker/README.md); permission denied/iOS fallback/410/revoke/logout, no PII payload | Đủ nền |
| 6. Hủy/giao thất bại/return/refund | [Flows § FLOW-04/06](../system/business-flows.md), [Returns § State](../domains/returns-refunds.md) | [Data § order/payment/return](../system/data-model.md), [API § 6–7](../system/api-contracts.md) | [Payment § Attempt](../domains/payment-simulation.md), [Communications § Taxonomy](../domains/communications.md); cancel-vs-ship, duplicate refund, fee once; **failed delivery thiếu** | Một phần: finding H-02/M-02 |
| 7. Shop mới báo người theo dõi | [Wishlist § Contract](../domains/wishlist.md), [Communications § Campaign](../domains/communications.md) | [API § 8](../system/api-contracts.md); follow data/API chưa có | [Worker § fanout](../apps/worker/README.md); unpublished, unsubscribe, shop suspend | Chưa cho follower F1; F2 Proposed |
| 8. Admin gửi announcement phân khúc | [Flows § FLOW-08](../system/business-flows.md), [Communications § Campaign](../domains/communications.md) | [Data § campaign](../system/data-model.md), [API § 8](../system/api-contracts.md) | [Worker § fanout](../apps/worker/README.md); second approver critical, consent, batch retry, quiet hours | Một phần: H-04/M-03 |
| 9. Seller self-register/được duyệt/đăng bán | [Marketplace § 2–4](../domains/marketplace-seller.md), [Catalog § Proposal](../domains/catalog.md) | [Data § shops/proposal](../system/data-model.md), [API § 9–12](../system/api-contracts.md) | [Communications § Marketplace](../domains/communications.md); invite replay, last owner, matching error | Đủ nền contract; phase Open M-01 |
| 10. Seller tạo ad/trả tiền/xem hiệu quả | [Advertising § Placement/Targeting](../domains/advertising.md), [Ad billing § Aggregate](../domains/advertising-billing.md) | [Data § F3 entities](../system/data-model.md), [API § 11](../system/api-contracts.md) là blueprint | Ad event/receipt/reconcile **dự kiến**; price model/provider/refund chưa duyệt | Chưa, F3 gated |
| 11. Buyer thấy sponsored search result | [Advertising § Placement](../domains/advertising.md), [Design § Sponsored](../system/design-system.md) | [API § 11](../system/api-contracts.md), [Discovery § ranking](../domains/discovery.md) | Impression/click dự kiến; organic sort independence, disclosure, suspended listing | Một phần, F3 gated |
| 12. Buyer bấm affiliate sang partner | [Affiliate § Hai hành trình/Luồng](../domains/affiliate.md), [Design § Sponsored](../system/design-system.md) | [Data § F3 affiliate](../system/data-model.md), [API § 11](../system/api-contracts.md) là blueprint | Outbound click dự kiến; allowlist, consent, expired offer, deep link fallback, **không tạo Cartasy order** | Một phần, F3 gated |
| 13. Ghi doanh thu quảng cáo/affiliate thật | [Reporting § Ranh giới](../domains/reporting.md), [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md) | [Data § F3 billing/affiliate](../system/data-model.md), [Ad billing § Gate](../domains/advertising-billing.md) | Receipt/partner report/reversal/reconciliation dự kiến; mock order value không được cộng | Chưa, F3 gated |

Các race/security bổ sung: concurrent quote/stock/quota; same/different idempotency key; parent cancel vs child ship; pending timeout vs delayed success; partial return và shipping fee one-time; outbox replay; SMTP/Web Push accepted-before-crash; session/role revoke ngay trước send; seller cross-shop ID/export; admin self-grant; affiliate open redirect/SSRF/rights expiry; partner report giả/duplicate/reversal; ad click fraud, overspend, invoice/provider mismatch và Finance role separation. [Testing § Ma trận/Gate](../platform/testing-quality.md), [Security § 8–9](../system/security-privacy.md), [Affiliate § Gate](../domains/affiliate.md).

## 5. Monetization readiness và nghiên cứu đối tác

| Trục | Đã có trong docs | Chưa có để bật thật | Mức |
|---|---|---|---|
| Sponsored serving | Placement candidates, disclosure, moderation, targeting/fraud principles. [Advertising § Placement](../domains/advertising.md) | Slot catalog, density, auction/fixed selection, exact API/DTO, billing trigger, contracts/test. | R1 boundary |
| Ad billing thật | Owner riêng, invoice/receipt/ledger/reconcile/gate. [Ad billing § Aggregate/Gate](../domains/advertising-billing.md) | Product/Finance/Legal chốt pricing, entity/tax/invoice/provider/refund/dispute/recognition; sandbox and controls. | R0 implementation |
| Affiliate outbound | External offer tách listing, CTA/redirect/consent, partner report proof. [Affiliate § Hai hành trình/Gate](../domains/affiliate.md) | Partner account/acceptance, quyền asset/link/API, market-specific terms, feed/report schema, payout/settlement. | R1 boundary; R0 integration |
| Seller self-service ads | Capability/module direction. [Advertising § Placement](../domains/advertising.md), [Admin workspace § F3](../apps/admin/README.md) | Advertiser account verification, roles, forms, review SLA, commercial agreement. | R0 implementation |

Nguồn **chính thức** được kiểm tra ngày 19/10/2026: [Shopee Affiliate VN terms](https://shopee.vn/affiliate/dieu-kien-hop-tac-chuong-trinh-tiep-thi-lien-ket-cua-shopee-viet-nam-update-18-08-2023/) và [Lazada Affiliate VN terms](https://pages.lazada.vn/wow/i/vn/corp/affiliate-terms) cho thấy điều kiện chương trình/đơn vị tham gia và commission do đối tác quy định; [TikTok Shop Affiliate Links — US](https://seller-us.tiktok.com/university/essay?knowledge_id=4049972027557678) mô tả cơ chế link ngoài ở **US**, không xác nhận quyền tại Việt Nam. [Lazada Open Platform app limits](https://open.lazada.com/apps/doc/doc?docId=108055&nodeId=10433) có quota/trạng thái App Key chung nhưng **không chứng minh Affiliate API được cấp**. Đây là due diligence sơ bộ, không là approval, hợp đồng hay hạn mức API của Cartasy; lưu bản terms hiện hành trong hồ sơ partner trước triển khai. Không chọn một đối tác, tỷ lệ commission hoặc pricing model bằng suy đoán.

## 6. MVP vs Future Roadmap

**P0/F0–F1 bắt buộc:** multi-seller đang hoạt động, bốn process Web/Admin chung/API/Worker, auth/MFA/tenant, approved catalog/listing/media/stock, search/filter/product/shop, guest/server cart, per-shop quote/voucher, atomic mixed-shop checkout, mock payment/ship/refund/order, notification transactional, buyer/seller/platform-admin scoped operations, moderation/support, security/data/observability/recovery gates. Buyer không trả tiền và không có ad billing/affiliate outbound ở F1. [Scope § 4–7](../system/product-scope.md), [Roadmap § F0–F1](roadmap.md). Self-registration phase vẫn là Open M-01; nếu hoãn, cần một cách tạo hai shop Seller F1 có audit.

**P1/F1 gate đã duyệt hoặc F2 growth:** F1 hiện đã nhận CMS, Web Push/SSE, campaign, review/return/support/reporting; đề xuất chia beta F1a/F1b chỉ sau Product quyết định, không âm thầm hạ scope. F2 flash sale, rule-based recommendation/bulk import; autocomplete, follow, recently viewed, gamification không quy đổi tiền là Proposed. [Roadmap § F1–F2](roadmap.md), [Feature map](../system/feature-map.md).

**P2/F3 có điều kiện:** sponsored serving, advertiser self-service, real ad billing, partner affiliate outbound/real commission, personalized recommendation. Gate ad serving, ad billing và partner affiliate **riêng**; không cần dựng ad auction, wallet, payout Seller, native app hoặc real Cartasy purchase chỉ để đáp ứng hướng doanh thu. [Roadmap § F3](roadmap.md), [Ad billing § Gate](../domains/advertising-billing.md), [Affiliate § Gate](../domains/affiliate.md).

## 7. Open Decisions

| Quyết định/owner | Câu hỏi cần chốt trước code/gate | Tác động |
|---|---|---|
| Open Decision M-01 Product | Self-registration shop giữ F1 hay F2? Nếu hoãn, Admin provisioning và Terms/MFA cho hai Seller F1 ra sao? | Scope Seller/API/UI/test; [Open decisions](audit-and-open-decisions.md) |
| O-01 Product/Ops | Parent order status khi child cancelled/returned/delivered trộn; ai đóng order và khi nào? | H-01; [Orders](../domains/orders.md) |
| O-02 Product/Ops | Seller hay worker chuyển mỗi bước shipping; giao thất bại có state/retry/return/refund/notification nào? | H-02; [Shipping](../domains/shipping-simulation.md) |
| O-03 Product/Pricing | Voucher stacking, quota restore, partial cancel, ship fee refund và rounding cuối cùng? | M-02; [Promotions](../domains/promotions-vouchers.md) |
| N-01 Product/Privacy | Follow shop đưa vào F2 hay F1; consent category/channel, quiet hours/frequency cap và inbox marketing cụ thể? | H-04/M-03; [Communications](../domains/communications.md) |
| Open Decision M-02/M-03 Product/Finance/Legal | Ad inventory đầu, advertiser eligibility, CPC/CPM/fixed, pre/postpay, tax/invoice/provider/refund/revenue policy? | Finding H-05; [Advertising](../domains/advertising.md), [Ad billing](../domains/advertising-billing.md) |
| Open Decision M-04/M-05 Product/Partnerships/Privacy | Partner đầu tiên/quốc gia/chương trình/quyền link/feed/report, consent-off redirect, external offer cạnh canonical product? | Finding H-06; [Affiliate](../domains/affiliate.md) |
| L-01 Product/Legal/Privacy/Ops | Domain, media rights, Privacy/Terms, retention, support SLA, SLO/traffic/provider/on-call/restore sign-off? | Public launch; [Audit cũ § Điểm chặn](audit-and-open-decisions.md), [Operations § 4–7](../platform/deployment-operations.md) |

Đây là **câu hỏi cần xác nhận**, không phải lời mời bật tiền thật hoặc thay đổi contract ngầm. Những giả định A-01–A-09 của [Open decisions](audit-and-open-decisions.md) vẫn có hiệu lực cho đến khi owner chốt hoặc cập nhật theo quy trình.

## 8. Implementation Readiness

`R0` = chưa đủ business decision/contract để code feature; `R1` = đủ định hướng scaffold/vertical slice có giới hạn; `R2` = spec owner+API/data/test oracle đủ để code end-to-end; `R3` = đã code/gate có bằng chứng. **Không có R3**; các mức dưới đây chỉ đánh giá docs sau cập nhật, không chứng nhận implementation.

| Module | Mức | Điều kiện nâng |
|---|---|---|
| F0 workspace/source boundary/DB/auth/outbox | R1 | Manifest/lock/peer, schema/migration, provider auth snapshot, CI/test trên PG thật. [Source architecture § 2–7](../system/source-architecture.md), [Local § bootstrap](../platform/local-development.md). |
| Catalog/listing/search/inventory | R1 | Proposal DTO/examples, popular ranking, upload/rights acceptance, race test. [Catalog § ProductProposal](../domains/catalog.md), [Discovery § Contract](../domains/discovery.md). |
| Seller lifecycle/portal | R1, phase self-registration Open | Chốt M-01, activation/invite/owner transfer examples/security test. [Marketplace § 2–4](../domains/marketplace-seller.md). |
| Cart/quote/checkout/payment mock | R1 | Per-shop selection vừa chốt, cần OpenAPI/fixture; promotion allocation/timeout/rollback/concurrency oracle. [Cart § Quote/Place](../domains/cart-checkout.md), [API § 5–6](../system/api-contracts.md). |
| Orders/fulfillment/returns | **R0** | H-01/H-02/O-03, parent status truth table, shipment failure and partial refund/cancel races. [Orders § Permission](../domains/orders.md), [Shipping § Quy trình](../domains/shipping-simulation.md). |
| Transactional notifications | R1 | Ma trận event→recipient/channel/link đã có; còn template DTO, route security/account, provider duplicate semantics test. [Communications § 3](../domains/communications.md). |
| Marketing/follower notifications | R0 cho follower; R1 cho opt-in chung | Chốt N-01, follow data/API nếu đưa vào release. [Communications § 4](../domains/communications.md). |
| CMS/reviews/support/risk/reporting | R1 | DTO/window/SLA/appeal/privacy/lag gate, actual test artifacts. [CMS § Contract](../domains/cms-content.md), [Returns § State](../domains/returns-refunds.md), [Risk § API](../domains/risk-moderation.md). |
| Ad serving/affiliate boundary | R1 design, R0 implementation | M-02/M-04, approved placement/partner contracts, API/schema and test. [Advertising](../domains/advertising.md), [Affiliate](../domains/affiliate.md). |
| Real ad billing/partner revenue | **R0** | Finance/Legal/partner/provider, ledger/invoice/report/settlement and independent release gates. [Ad billing](../domains/advertising-billing.md), [Affiliate § Partner due diligence](../domains/affiliate.md). |

## 9. Thay đổi đã làm và giới hạn audit

Các file sở hữu hiện hành theo hướng **scope → ADR → domain → data/API → app/platform → delivery**: [Product scope](../system/product-scope.md), [Roadmap](roadmap.md), [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md), [ADR-0011](../decisions/ADR-0011-unified-admin-seller-workspace.md), [Advertising](../domains/advertising.md), [Ad billing](../domains/advertising-billing.md), [Affiliate](../domains/affiliate.md), [Data model](../system/data-model.md), [API contracts](../system/api-contracts.md), [Security](../system/security-privacy.md), [Design](../system/design-system.md), [Reporting](../domains/reporting.md), [Integrations](../platform/external-integrations.md) và các app/test/operations/checklist liên quan. Các contract F1 rõ hơn gồm per-shop shipping, activation/invite, proposal, Admin user operations và notification archive.

Chưa chốt hay code các vấn đề **Open** trong §7; không có migration/OpenAPI/artifact thực tế, E2E, provider/partner approval hoặc legal/Finance sign-off. Checklist vẫn toàn `[ ]`. Trước khi bắt đầu một vertical slice, agent phải đọc owner spec, viết DTO/state matrix/acceptance test cho gap tương ứng, rồi mới tạo schema/API/UI/worker và lưu gate theo [Checklist § Definition of Done](build-checklist.md). Không tự ghi “đạt 100%”.

Rà soát cấu trúc hiện hành: mỗi file có `doc_id` duy nhất, front matter và code fence hợp lệ, không có Markdown link nội bộ trỏ tới file thiếu. Đây là kiểm tra tĩnh nội dung tài liệu; trạng thái contract/implementation vẫn theo các bảng ở trên.
