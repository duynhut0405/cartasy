---
doc_id: ADR-INDEX
title: Cartasy Architecture Decisions
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-20
version: 4.4.1
source_of_truth: Danh mục quyết định kỹ thuật có hiệu lực
review_cycle: Khi thêm/thay ADR
---

# Quyết định kiến trúc

Các ADR dưới đây là quyết định nền cho tài liệu hiện hành. ADR-0010 giữ ranh giới monetization; ADR-0011 chốt một app quản trị cho Seller và Platform Admin; ADR-0012 chốt storefront đa ngôn ngữ; ADR-0013 chốt frontend state/UI/tooling. Khi đổi quyết định đã accepted, cập nhật nguồn sở hữu và các contract liên quan trong cùng thay đổi.

| ADR | Quyết định |
|---|---|
| [0001](ADR-0001-modular-monolith.md) | Modular monolith và bounded context |
| [0002](ADR-0002-product-listing-shop.md) | Lịch sử: product/listing/shop, superseded bởi 0009 |
| [0003](ADR-0003-simulated-payment.md) | Thanh toán, order và giao nhận mô phỏng |
| [0004](ADR-0004-stack-and-source.md) | Node/Next/Nest/PostgreSQL/Drizzle và cấu trúc source |
| [0005](ADR-0005-transactional-outbox.md) | Transactional outbox và worker idempotent |
| [0006](ADR-0006-auth-and-tenant.md) | Better Auth và quyền theo shop |
| [0007](ADR-0007-portable-deployment.md) | Container Node portable, staging/production tách biệt |
| [0009](ADR-0009-multiseller-mvp.md) | Multi-seller hoạt động ngay F1, supersedes 0002 |
| [0010](ADR-0010-simulated-commerce-real-monetization-boundary.md) | Đơn Cartasy mô phỏng, ad billing/affiliate partner revenue có thể là tiền thật sau MVP, tách dữ liệu/gate |
| [0011](ADR-0011-unified-admin-seller-workspace.md) | Một `apps/admin` cho Seller và Platform Admin; RBAC + shop capability + ABAC ở API |
| [0012](ADR-0012-multilingual-storefront.md) | Web `/[locale]` dùng `next-intl`, nội dung dịch theo owner và public SEO theo bản published |
| [0013](ADR-0013-frontend-state-ui-tooling.md) | Zustand chỉ cho UI state dùng chung, shadcn/ui bản Radix + Tailwind, Prettier/Husky format source; vị trí source UI được 0014 thay thế |
| [0014](ADR-0014-app-owned-shadcn-components.md) | Web/Admin dùng shadcn trực tiếp trong từng app; chỉ tạo component riêng khi cần, không tạo `@cartasy/ui` ở F0/F1 |
