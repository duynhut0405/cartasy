---
doc_id: DOC-INDEX
title: Cartasy — bản đồ tài liệu chuẩn
status: accepted
owner: Product & Engineering
last_updated: 2026-10-20
version: 4.4.1
source_of_truth: Cấu trúc, quyền sở hữu và thứ tự đọc tài liệu
---

# Cartasy — Single Source of Truth

Cartasy là marketplace đa seller cho Buyer mua sắm miễn phí. **Đơn mua, thanh toán, vận chuyển và hoàn tiền trong Cartasy đều mô phỏng; không thu tiền Buyer hay giao hàng thật.** Giá hiển thị bằng VND, không dùng credit/điểm làm đơn vị thanh toán. Sau MVP, quảng cáo có thể thu tiền thật từ Advertiser và affiliate có thể dẫn Buyer sang đối tác để mua thật; hai dòng này có module, chứng từ và gate riêng, không đổi mock checkout. Seller và Platform Admin dùng chung một `apps/admin`, quyền được kiểm tại API. [Product scope](system/product-scope.md), [ADR-0010](decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md), [ADR-0011](decisions/ADR-0011-unified-admin-seller-workspace.md). Tài liệu là **đích thiết kế**, không khẳng định source code đã triển khai.

## Cây tài liệu

Web F1 có route `/[locale]` với `vi`/`en` và `next-intl`; nội dung catalog/shop/CMS theo bản dịch đã publish. Web/Admin dùng Zustand cho UI state dùng chung, shadcn/ui bản Radix trực tiếp trong từng app + Tailwind cho component, Prettier/Husky cho format source. [Web guide](apps/web/README.md), [ADR-0012](decisions/ADR-0012-multilingual-storefront.md), [ADR-0013](decisions/ADR-0013-frontend-state-ui-tooling.md), [ADR-0014](decisions/ADR-0014-app-owned-shadcn-components.md).

```text
docs/
  README.md                         # bản đồ và thứ tự ưu tiên
  system/                           # quy tắc chung, chỉ viết một lần
    product-scope.md                 # mục tiêu, persona, scope, điều kiện nghiệm thu
    feature-map.md                   # danh mục module và giai đoạn
    business-flows.md                # hành trình xuyên module
    architecture.md                  # context, giao tiếp, transaction, event
    data-model.md                    # quan hệ, khóa, bất biến dữ liệu
    api-contracts.md                 # endpoint inventory và HTTP contract
    security-privacy.md              # authn/authz/RBAC, session, dữ liệu cá nhân
    source-architecture.md           # workspace, package boundary và dependency
    design-system.md                 # UI token, accessibility, content
    analytics.md                     # event/KPI/consent
    catalog-governance.md            # chuẩn nội dung và moderation
  domains/                           # mỗi file sở hữu quy tắc nghiệp vụ module, gồm trust & safety
  apps/
    source-preview.md                # thư viện, module và state/store theo từng src
    web/README.md                    # storefront
    admin/README.md                  # một app Seller + Platform Admin
    api/README.md                    # HTTP/composition root
    worker/README.md                 # outbox và scheduler
  platform/
    stack-dependencies.md            # phiên bản và thư viện chuẩn
    external-integrations.md         # SMTP/push/SSE/webhook và adapter
    local-development.md             # bootstrap workspace và service local
    database-migrations.md           # schema/migration/seed
    testing-quality.md               # test và release gate
    deployment-operations.md         # môi trường, vận hành, recovery
  packages/README.md                # shared packages và exports
  delivery/
    audit-and-open-decisions.md      # audit, giả định, vấn đề mở
    full-product-audit-2026-10-19.md  # audit hiện hành, coverage và readiness
    roadmap.md                       # giai đoạn, exit gate
    build-checklist.md               # triển khai và bằng chứng
  decisions/                         # ADR đã chấp thuận
  templates/                         # mẫu tài liệu/ADR
```

## Quyền sở hữu nội dung

| Câu hỏi | Nguồn chuẩn |
|---|---|
| Sản phẩm làm gì, F1/F2/F3 và tiền thật tách thế nào? | [Product scope](system/product-scope.md), [feature map](system/feature-map.md), [roadmap](delivery/roadmap.md), [ADR-0010](decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md) |
| Trạng thái, validation, lỗi và quyền của một module? | File tương ứng trong [domains](domains/README.md) |
| Luồng nhiều module, transaction, outbox? | [Business flows](system/business-flows.md), [architecture](system/architecture.md) |
| Bảng, khóa, index, bất biến dữ liệu? | [Data model](system/data-model.md) và [database migrations](platform/database-migrations.md) |
| HTTP path, DTO, status/error code? | [API contracts](system/api-contracts.md) |
| Role, session, tenant, privacy? | [Security & privacy](system/security-privacy.md) |
| Phiên bản, dependencies, dependency direction? | [Stack](platform/stack-dependencies.md), [source architecture](system/source-architecture.md), [shared packages](packages/README.md) |
| Cách setup, tổ chức `src`, locale route và UI composition? | [Web](apps/web/README.md), [Admin chung Seller/Platform](apps/admin/README.md), [API](apps/api/README.md), [Worker](apps/worker/README.md), [Design system](system/design-system.md) |
| Xem nhanh thư viện, module và state/store theo từng `src`? | [Source preview](apps/source-preview.md); chi tiết vẫn ở app guide/stack tương ứng |
| Chạy, test, release? | [Local development](platform/local-development.md), [integrations](platform/external-integrations.md), [testing](platform/testing-quality.md), [operations](platform/deployment-operations.md) |

Khi tài liệu có vẻ khác nhau: yêu cầu sản phẩm được chấp thuận mới nhất → ADR `accepted` cho quyết định kỹ thuật → **file sở hữu** trong bảng → tài liệu triển khai của app. Tài liệu app chỉ ánh xạ quy tắc chung thành code/UI, không tự đặt thêm chính sách nghiệp vụ. Code hiện trạng là bằng chứng triển khai, không tự sửa đặc tả. Khi đổi một contract, sửa file sở hữu trước, rồi cập nhật app, OpenAPI, migration/test và checklist liên quan trong cùng thay đổi.

## Lộ trình đọc cho AI Agent

1. Đọc trang này, [scope](system/product-scope.md), [feature map](system/feature-map.md) và [roadmap](delivery/roadmap.md).
2. Đọc [architecture](system/architecture.md), [source architecture](system/source-architecture.md), [stack](platform/stack-dependencies.md), [data model](system/data-model.md), [API contracts](system/api-contracts.md) và [security](system/security-privacy.md).
3. Đọc [module nghiệp vụ](domains/README.md) của vertical slice sắp xây; xem [audit và quyết định mở](delivery/audit-and-open-decisions.md) để không biến giả định thành yêu cầu.
   Với monetization, đọc [audit sản phẩm hiện hành](delivery/full-product-audit-2026-10-19.md) và [ADR-0010](decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).
4. Đọc [hướng dẫn app](apps/README.md), [local development](platform/local-development.md), [migrations](platform/database-migrations.md), [testing](platform/testing-quality.md).
5. Làm theo [checklist](delivery/build-checklist.md); chỉ đánh dấu hoàn thành khi có bằng chứng chạy.

Mọi trang tài liệu phải có `doc_id` duy nhất, `status`, `owner`, `last_updated` và `version`. Không dùng tài liệu cũ ngoài cây này làm nguồn chuẩn. Các ví dụ tên file/câu lệnh là **blueprint cần tạo**; không mặc định file đó đang tồn tại.
