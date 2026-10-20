---
doc_id: DOC-CHANGELOG
title: Cartasy Documentation Baseline
status: accepted
owner: Cartasy Product & Engineering
last_updated: 2026-10-20
version: 4.4.1
source_of_truth: Tóm tắt bộ tài liệu chuẩn hiện hành
review_cycle: Khi có thay đổi phạm vi hoặc quyết định nền
---

# Bộ tài liệu hiện hành — 4.4.1

- Web/Admin dùng shadcn/ui bản Radix trực tiếp trong từng app; không tạo `@cartasy/ui` ở F0/F1. Chỉ compose component riêng khi shadcn thiếu nhu cầu thực tế. [ADR-0014](decisions/ADR-0014-app-owned-shadcn-components.md).

- Web/Admin chọn Zustand cho UI state dùng chung theo feature/widget; React Query giữ API state và URL giữ state có thể chia sẻ. Component nền dùng shadcn/ui bản Radix, style bằng Tailwind/token; Prettier + Husky/lint-staged format source, CI kiểm `format:check`. [ADR-0013](decisions/ADR-0013-frontend-state-ui-tooling.md).

- Marketplace đa seller ngay F1. Buyer dùng Web; Seller và Platform Admin dùng chung **một `apps/admin`** với khu `/seller/*` và `/platform/*`. API kiểm RBAC, shop capability và ABAC trên từng tài nguyên. Backend gồm API và Worker.
- Web F1 dùng `next-intl` với route `/[locale]` (`vi`/`en`); UI/design structure, nội dung catalog/shop/CMS, API/cache/SEO và notification theo locale. [ADR-0012](decisions/ADR-0012-multilingual-storefront.md).
- [Source preview](apps/source-preview.md) gom thư viện, module `src`, state/store và tooling của bốn app cùng shared packages để xem trước khi scaffold.
- Checkout, đơn hàng, payment, shipping và refund trong Cartasy đều mô phỏng. Buyer không trả tiền hoặc nhận hàng thật. Giá VND do server quyết định.
- Sau MVP, quảng cáo có thể thu tiền thật từ Advertiser và affiliate có thể chuyển Buyer sang đối tác mua thật; hai luồng có domain, chứng từ, đối soát và gate riêng, không dùng mock order làm doanh thu.
- [ADR-0010](decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md) và [ADR-0011](decisions/ADR-0011-unified-admin-seller-workspace.md) sở hữu hai ranh giới trên. [Audit sản phẩm](delivery/full-product-audit-2026-10-19.md) ghi độ phủ, điểm mở và mức sẵn sàng.
- Đây là blueprint tài liệu. Chưa có source ứng dụng, migration, OpenAPI artifact hoặc kết quả chạy gate; [build checklist](delivery/build-checklist.md) lưu bằng chứng khi triển khai.
