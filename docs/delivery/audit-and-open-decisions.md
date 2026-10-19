---
doc_id: DOC-AUDIT
title: Documentation Audit and Open Decisions
status: accepted
owner: Product & Architecture
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Kết quả audit, mức phủ tài liệu, quyết định chưa xác minh
---

# Audit tài liệu và quyết định mở

## Phương pháp và giới hạn

Đọc toàn bộ tài liệu trong `docs`, đối chiếu product scope → domain state → schema → API → app → vận hành/test. “Đã tài liệu hóa” chỉ nghĩa là có đặc tả nhất quán; **không** có nghĩa app đã tồn tại hoặc gate đã chạy. Các số version/package phải được kiểm lại peer dependency và lockfile khi scaffold. Câu hỏi sản phẩm chưa được trả lời được ghi rõ thay vì tự coi là yêu cầu.

## Phát hiện và cách xử lý

| Phát hiện đã khép | Tác động | Quy tắc hiện hành |
|---|---|---|
| F1 chỉ một platform shop, trái mục tiêu marketplace đa seller | Kiến trúc/roadmap/UI/API không thực thi marketplace ở MVP | Chốt multi-seller F1 theo xác nhận của chủ dự án; cập nhật scope/roadmap/flow/app/schema/API/permission/gate |
| Notification chỉ có order/email, thiếu product/promotion/system announcement và browser push | Không đáp ứng trải nghiệm “bắn noti” | Đặc tả campaign fanout, preference, Web Push opt-in, SSE, SMTP, delivery, dedupe/quiet hours ở [Notifications](../domains/communications.md) |
| CMS, reports, flash sale, recommendation, affiliate thiếu hoặc chỉ nêu tên | Agent tự suy feature boundary | Tạo domain spec và phân loại F1/F2/F3 |
| Payment webhook/real carrier/FCM/queue/realtime/cache chưa có quyết định ranh giới | Dễ tích hợp thật trái phạm vi mô phỏng hoặc chọn hạ tầng thừa | Ghi adapter/trigger/gate trong [Integrations](../platform/external-integrations.md), [Architecture](../system/architecture.md) |
| Seller onboarding, MFA, tenant data, status và API còn sơ lược | Cross-tenant/role leak | Làm rõ khu Seller trong Admin workspace và shop policy từ MVP; test hai shop/hai buyer |
| Một số domain spec chỉ có vài đoạn, thiếu state/permission/error/contract map | Khó triển khai độc lập | Bổ sung flow, state, quyền, API/data map và test theo [domain index](../domains/README.md) |
| Marketplace thiếu tuyến report/takedown/appeal và quyền seller ở hậu mãi | Dễ để lọt nội dung vi phạm hoặc lộ dữ liệu giữa shop | Thêm [Trust & Safety](../domains/risk-moderation.md), seller reply/proposal và admin decision có audit |
| Stack/setup/app hướng dẫn lặp và thiếu shared package ownership | Dễ tạo src tùy tiện | [Source architecture](../system/source-architecture.md) và [Shared Packages](../packages/README.md) sở hữu ranh giới; app guides sở hữu cây src |
| Giả định kỹ thuật lẫn với quyết định sản phẩm | Agent có thể triển khai điều chưa được duyệt | Phân loại bên dưới, ghi exit gate xác minh |

## Quyết định đã xác nhận

| Mã | Nội dung | Nguồn |
|---|---|---|
| C-01 | Marketplace đa seller hoạt động trong MVP | Xác nhận của chủ dự án ngày 19/10/2026 |
| C-02 | Payment, fulfillment và refund mô phỏng; không tiền/hàng thật | Xác nhận trước đó của chủ dự án |
| C-03 | Không lấy credit/điểm làm đơn vị thanh toán | Yêu cầu sản phẩm |
| C-04 | Bộ `docs` là nguồn chuẩn mới, không dùng nội dung không liên quan | Yêu cầu sản phẩm |
| C-05 | Buyer dùng luồng mua sắm mô phỏng Cartasy miễn phí; không thu tiền Buyer/giao hàng thật | Yêu cầu mới ngày 19/10/2026, [Scope](../system/product-scope.md) |
| C-06 | Sau MVP Cartasy hướng tới **doanh thu quảng cáo thật** từ advertiser và **hoa hồng affiliate thật** từ đối tác; Buyer có thể rời Cartasy mua thật ở đối tác, không tạo Cartasy order thật | Yêu cầu mới ngày 19/10/2026, [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md) |
| C-07 | Seller và Platform Admin dùng chung một `apps/admin`; quyền theo RBAC + shop capability + ABAC | Xác nhận của chủ dự án ngày 19/10/2026, [ADR-0011](../decisions/ADR-0011-unified-admin-seller-workspace.md) |
| C-08 | Web có route `[locale]` và dùng `next-intl`; UI đa ngôn ngữ, nội dung và thiết kế có cấu trúc rõ | Yêu cầu của chủ dự án ngày 19/10/2026, [ADR-0012](../decisions/ADR-0012-multilingual-storefront.md) |

## Giả định kỹ thuật đề xuất, cần xác minh khi triển khai

| Mã | Giả định mặc định để agent có thể setup | Cần xác minh trước | Owner |
|---|---|---|---|
| A-01 | Việt Nam, VND, một region triển khai; UI Web `vi`/`en` đã chốt | Domain/thị trường vận hành đầu tiên | Product |
| A-02 | Một kho ảo mỗi listing; không chuyển hàng cho seller | Kịch bản seller có phải tự bấm trạng thái đơn hay scheduler tự chạy | Product/Ops |
| A-03 | Một voucher code/order và campaign tự động không chồng trên item | Chính sách stacking/return coupon | Product |
| A-04 | Web Push opt-in; in-app transactional bắt buộc; email theo loại/preference | Loại notification marketing, quiet hours và consent text | Product/Privacy |
| A-05 | Seller verification tối thiểu qua email/hồ sơ shop, không thu ngân hàng | Mức độ xác minh, điều khoản seller và quyền media | Product/Legal |
| A-06 | Support là ticket async, không live chat | Giờ phục vụ/SLA công khai | Ops |
| A-07 | PostgreSQL outbox/jobs + SSE trước; Redis/queue/WebSocket chuyên dụng khi có số đo | Traffic và SLO thực tế | Engineering |
| A-08 | CMS quản lý home/banner/page theo `vi`/`en`, không cho HTML tùy ý | Workflow biên tập và legal pages | Product/Content |
| A-09 | Review 30 ngày, return 7 ngày, seller chỉ đề xuất return, Admin quyết định; report/appeal qua support | Cửa sổ hậu mãi, reviewer độc lập và danh mục vi phạm áp dụng | Product/Ops/Legal |

Giả định không được viết thành điều khoản/giá/phí/hoa hồng thật. Agent triển khai mặc định kỹ thuật có thể thay đổi sau xác minh; thay đổi ảnh hưởng model phải cập nhật owner doc, ADR, migration/test.

## Quyết định mở

| Mã | Cần Product Owner/owner xác nhận | Vì sao chưa tự quyết |
|---|---|---|
| M-01 | Self-registration shop thuộc F1 hay F2? Nếu chuyển sang F2, hai shop active trong MVP được Admin cấp theo quy trình nào? | Phạm vi hiện tại chưa chốt phase; ảnh hưởng scope/API/gate Seller |
| M-02 | Ad product nào trước (Sponsored Product/Shop/Search/Display), vị trí, giới hạn mật độ, buyer disclosure và advertiser eligibility? | Cần xác minh nhu cầu/bán hàng trước khi chốt UI/slot |
| M-03 | Mô hình CPC/CPM/fixed, giá, ngân sách, prepay/postpay, overdelivery, refund/credit note, pháp nhân phát hành invoice và provider thanh toán quảng cáo? | Là quyết định thương mại, tài chính, pháp lý và security; không suy từ mock payment |
| M-04 | Đối tác affiliate/quốc gia/chương trình cụ thể; website/app Cartasy có được duyệt, quyền link/feed/API/report, attribution window, disclosure, conversion/commission settlement và tax? | Điều khoản khác theo đối tác/thị trường; chưa có tài khoản/hợp đồng/bằng chứng quyền |
| M-05 | Khi Buyer từ chối tracking, vẫn cho outbound link không attribution hay ẩn CTA; external offer có đặt cạnh canonical product không? | Tác động consent, UX và mô hình doanh thu |
| M-06 | Khi nào thử PWA/native app, notification native, deep link affiliate? | API/auth/token/push khác web hiện tại; không là gate F1 |

Các mục M-* là **Open Decision**, không phải policy đã phê duyệt. [Audit sản phẩm](full-product-audit-2026-10-19.md) lưu tác động và bằng chứng. Shopee/Lazada/TikTok Shop official terms cần kiểm lại lúc ký kết; quyền Open Platform chung không chứng minh quyền affiliate.

## Điểm chặn trước public launch

- Chốt domain, brand, email sender, object storage, hosting, secret manager, DNS/HTTPS, privacy/Terms/seller Terms và quyền sử dụng dữ liệu sản phẩm.
- Chốt retention áp dụng, consent text Web Push/marketing, account deletion, incident contact.
- Chốt SLO/traffic dự kiến, ngân sách, lịch hỗ trợ, chính sách hậu mãi và quy trình xử lý nội dung/abuse/appeal.
- Tạo code, migration, OpenAPI, CI/CD, staging/production, restore drill và bằng chứng kiểm thử. Không có nội dung nào ở đây khẳng định các bước đã hoàn tất.
- Trước bật monetization thật: provider/partner agreement, finance/legal/tax/privacy review, invoice/ledger/reporting boundary, consent/disclosure, reconciliation/incident/restore gate; không dùng mock order làm doanh thu.

## Checklist mức độ hoàn thiện tài liệu

| Phạm vi | Đặc tả | Bằng chứng triển khai |
|---|---|---|
| Scope/roadmap/quyết định | Đã đồng bộ multi-seller + mô phỏng | Chưa có |
| Domain F1 buyer/seller/admin | Có owner, state, validation, quyền, API/data map; cần review sản phẩm các giả định A-* | Chưa có |
| Notification transactional/campaign/realtime/push | Có kiến trúc/contract/permission/delivery | Chưa có |
| F2/F3 growth/monetization | Có boundary, điều kiện bật và invariant nền; chi tiết UX/kinh doanh chờ duyệt | Chưa có |
| App/shared packages setup | Có stack/`src`/env/command/gate | Chưa có |
| Security/operations/test | Có gate/runbook/baseline, cần SLO/provider thực | Chưa có |

Kiểm tra cấu trúc tài liệu hiện hành gồm `doc_id`, front matter, liên kết nội bộ, code fence và số cột bảng; kết quả phải được kiểm lại khi docs thay đổi. Đây là kiểm tra tài liệu, không thay thế content review/kiểm thử ứng dụng. [Audit sản phẩm](full-product-audit-2026-10-19.md) lưu các gap nghiệp vụ và mức sẵn sàng.

Checklist triển khai chi tiết ở [Build checklist](build-checklist.md).
