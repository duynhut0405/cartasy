---
doc_id: APP-ADMIN
title: Unified Admin Workspace — Seller and Platform
status: accepted
owner: Marketplace & Platform Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Setup, src, route, state và integration của apps/admin cho Seller/Platform Admin
---

# Admin workspace dùng chung

[Preview thư viện, module và state/store của toàn workspace](../source-preview.md).

## Vai trò và stack

`apps/admin` là **một** ứng dụng Next.js 16/React 19/TypeScript 6, một source tree/origin/deploy cho Seller và Platform Admin. Seller dùng `/seller/*`, nhân sự Cartasy dùng `/platform/*`. Một identity có thể có cả platform role và membership nhiều shop. Menu theo capability chỉ là UX; API/application layer quyết định RBAC + shop capability + ABAC (shop status, resource owner/state, MFA/recent auth) ở mỗi request. Admin không tự có quyền Seller và Seller không tự có quyền platform. [ADR-0011](../../decisions/ADR-0011-unified-admin-seller-workspace.md), [Security](../../system/security-privacy.md).

App dùng `@cartasy/ui`, Tailwind/Radix, `@tanstack/react-query`, `openapi-fetch`/`@cartasy/contracts`, `react-hook-form`/`zod`. Bảng lớn dùng semantic table trước; chỉ thêm grid virtualization khi profiling chứng minh cần. Không viết business rule/SQL/permission table trong Next.

## `src` chuẩn

```text
src/
  app/
    (auth)/sign-in/page.tsx
    (auth)/mfa/page.tsx
    (workspace)/page.tsx                   # redirect theo context được phép
    (seller)/seller/apply/page.tsx
    (seller)/seller/applications/[id]/page.tsx
    (seller)/seller/shops/page.tsx
    (seller)/seller/shops/[shopId]/page.tsx
    (seller)/seller/shops/[shopId]/profile/page.tsx
    (seller)/seller/shops/[shopId]/members/page.tsx
    (seller)/seller/shops/[shopId]/catalog/proposals/page.tsx
    (seller)/seller/shops/[shopId]/listings/page.tsx
    (seller)/seller/shops/[shopId]/inventory/page.tsx
    (seller)/seller/shops/[shopId]/orders/page.tsx
    (seller)/seller/shops/[shopId]/orders/[id]/page.tsx
    (seller)/seller/shops/[shopId]/promotions/page.tsx
    (seller)/seller/shops/[shopId]/notifications/page.tsx
    (seller)/seller/shops/[shopId]/notification-proposals/page.tsx
    (seller)/seller/shops/[shopId]/support/page.tsx
    (seller)/seller/shops/[shopId]/returns/page.tsx
    (seller)/seller/shops/[shopId]/reviews/page.tsx
    (seller)/seller/shops/[shopId]/reports/page.tsx
    (platform)/platform/page.tsx
    (platform)/platform/catalog/products/page.tsx
    (platform)/platform/catalog/products/[id]/page.tsx
    (platform)/platform/catalog/categories/page.tsx
    (platform)/platform/shops/page.tsx
    (platform)/platform/shop-applications/page.tsx
    (platform)/platform/cms/page.tsx
    (platform)/platform/cms/[id]/page.tsx
    (platform)/platform/notification-campaigns/page.tsx
    (platform)/platform/notification-campaigns/[id]/page.tsx
    (platform)/platform/reports/page.tsx
    (platform)/platform/listings/page.tsx
    (platform)/platform/inventory/page.tsx
    (platform)/platform/promotions/page.tsx
    (platform)/platform/orders/page.tsx
    (platform)/platform/orders/[id]/page.tsx
    (platform)/platform/returns/page.tsx
    (platform)/platform/reviews/page.tsx
    (platform)/platform/moderation/page.tsx
    (platform)/platform/support/page.tsx
    (platform)/platform/notifications/page.tsx
    (platform)/platform/users/page.tsx
    (platform)/platform/audit/page.tsx
    (platform)/platform/operations/page.tsx
    layout.tsx
    error.tsx
    loading.tsx
    not-found.tsx
  features/
    seller-onboarding/
    shop-switcher/
    shop-profile/
    membership/
    product-proposals/
    seller-listings/
    seller-inventory/
    seller-orders/
    seller-promotions/
    seller-notification-proposals/
    seller-support-returns-reviews/
    seller-reporting/
    catalog-management/
    listing-management/
    inventory-management/
    promotion-management/
    order-operations/
    return-operations/
    review-moderation/
    risk-moderation/
    support-operations/
    shop-governance/
    shop-onboarding-review/
    cms-management/
    notification-campaigns/
    reporting/
    access-management/
    operations-monitoring/
  entities/
    product/
    listing/
    shop/
    order/
    ticket/
    actor/
  widgets/
    workspace-shell/
    seller-shell/
    platform-shell/
    filter-toolbar/
    audit-diff/
    translation-editor/             # locale tabs, diff, preview, publish status
  shared/
    api/
    realtime/
    auth/
    config/
    ui/
    lib/
    styles/
```

`app` compose route; `features` chứa form/command UI; `entities` là presentation models; `widgets` layout/use-case composition; `shared` không chứa business rule. Không import source app khác. Seller/Platform route có feature riêng khi action/policy khác; primitive UI chỉ chia sẻ khi thực sự giống. Các màn hình rủi ro (publish, inventory adjust, refund simulation, account/shop suspension) buộc hiển thị trạng thái hiện tại, version, tác động, ô lý do và kết quả/audit ID.

## Thiết kế màn hình và biên tập đa ngôn ngữ

Admin có một design system từ `@cartasy/ui`: `workspace-shell` giữ navigation, breadcrumb, responsive sidebar, global feedback; Seller/Platform shell đổi menu và context theo capability, không tạo app hay UI kit thứ hai. Route chỉ compose widget/feature; bảng, filter toolbar, form, audit diff và publish checklist có loading/empty/error/forbidden/version-conflict state. Detail page tách summary, tabs, action panel và history; hành động nhạy cảm có reason, version và confirmation rõ. `shared/ui` chứa adapter primitive, domain component ở `features`/`entities`. [Design system](../../system/design-system.md).

Một workspace này quản lý bản dịch catalog/shop/CMS theo `vi`/`en`: editor hiển thị từng locale, source/revision/diff, preview Web route tương ứng, field thiếu và trạng thái draft/review/published. Publish theo quyền owner và review policy ở API; nút UI không cấp quyền. Seller chỉ đề xuất/sửa nội dung shop mình, platform reviewer duyệt canonical/CMS; không copy bản `vi` thành `en` published. Admin UI có thể dùng `next-intl` không route prefix theo user preference khi triển khai bản dịch Admin; URL `/seller/*` và `/platform/*` giữ ổn định, không cần hreflang/SEO. [ADR-0012](../../decisions/ADR-0012-multilingual-storefront.md).

## Routing, state, API

Toàn bộ route là private. App shell xác thực session/MFA rồi gọi `GET /v1/portal/context` lấy platform capabilities và shop memberships **hiện hành** để dựng menu/redirect; response chỉ phục vụ UX, không thay authorization của API nghiệp vụ. Seller chưa có shop có thể đi `/seller/apply` khi self-registration bật. `/platform/*` yêu cầu platform capability; `/seller/shops/:shopId/*` yêu cầu membership/capability/shop status. Identity có cả hai role chuyển workspace rõ ràng, không tự nâng quyền.

Session API cookie host-only nên data load ở client sau hydration với `credentials: include`; không SSR/cache private response từ Next origin. React Query key gồm actor ID + workspace kind + platform role/capability version hoặc shop ID/membership version + resource/filter/cursor. Khi logout, role/membership revoke hoặc đổi workspace/shop: cancel request/stream cũ, clear cache/form nhạy cảm, fetch lại context. Không giữ authority trong localStorage. Filter/sort/cursor trên URL; mutation dùng `If-Match` và idempotency key. `401` sign-in, `403` denied, `404` cross-shop, `409` reload version/diff, `423` suspended shop read-only. Không optimistic update cho payment, stock, order, return hoặc quyền.

Typed client trong **cùng app** gọi `/v1/seller/*` ở khu Seller và `/v1/admin/*` ở khu Platform. Không dùng endpoint Admin để né shop policy hoặc endpoint Seller để cấp quyền platform. Seller SSE `/v1/seller/shops/:shopId/stream`; Platform SSE `/v1/admin/stream`; hint dẫn tới refetch API owner, polling fallback. Dữ liệu nhạy cảm render tối thiểu; export cần capability riêng, giới hạn cột/audit/signed URL ngắn. UI không gửi raw PII vào logs/analytics. Seller chỉ nhận shop-order/inbox đúng shop. Admin tạo/duyệt CMS/announcement, campaign critical có second approver, xem delivery report aggregate, duyệt proposal, vận hành worker/audit. Không cho arbitrary recipient list; shop onboarding và hai-shop isolation là gate bắt buộc. [Catalog governance](../../system/catalog-governance.md), [Reviews](../../domains/reviews.md).

**Seller:** application draft/submit/needs changes; activation; owner invite/revoke/transfer; profile; product proposal, listing/price/stock; shop order transition mô phỏng; promotion, return proposal, review/support reply, notification proposal và shop aggregate report. Seller không sửa canonical product hoặc mock payment outcome, không xem địa chỉ Buyer đầy đủ. Shop switch hủy state/request của shop trước. Seller F1 dùng inbox/email; Web Push Seller không mặc định bật.

**Platform Admin:** user/role/shop governance; application/listing/product/content moderation; CMS; inventory/promotion/order/return/support operations; campaign approval/delivery report; analytics/audit/worker operations. Capability tách theo nhóm; action nhạy cảm cần MFA/recent auth, reason, version/audit. Platform operator xem shop không trở thành member; hành động thay shop chỉ qua Admin use case riêng có quyền và reason.

## Env và setup độc lập

`NEXT_PUBLIC_API_URL=http://localhost:3002/v1`, `NEXT_PUBLIC_ADMIN_URL=http://localhost:3001`, `PORT=3001`; không có `SELLER_URL`, cổng 3003, origin Seller riêng hoặc database/SMTP secret trong app. Admin origin nằm trong API CORS allowlist; production dùng HTTPS same-site sibling domain.

1. Pin Node/pnpm và cài workspace theo [local development](../../platform/local-development.md); migrate/seed hai seller/hai shop, platform admin và identity có cả hai role; bootstrap role bằng command an toàn, bật API/worker.
2. Tạo `apps/admin/package.json`/Next App Router, cây `src` và scripts `dev/build/start/lint/typecheck/test`; dùng package `@cartasy/contracts`/`ui` đã build.
3. Cấu hình `.env.local`; chạy `pnpm --filter @cartasy/admin dev` tại port 3001. Login Seller/Admin, enroll MFA, kiểm tra `/portal/context`, `/seller/*`, `/platform/*` và denied.
4. Smoke hai Seller chỉ thấy shop mình; Admin đúng capability duyệt application/proposal; identity hai role chuyển workspace/shop; revoke/suspend/stream reconnect không lộ cache cũ. Gate `pnpm --filter @cartasy/admin lint`, `typecheck`, `build` và E2E two-shop/two-buyer, cross-role deep link/API denial, CSRF, reason/audit, mobile accessibility.

Quy trình thay đổi: cập nhật policy/domain/API và audit schema trước, sinh type, xây feature theo module, test capability denial/cross-shop/cross-role, rồi review accessibility/empty/error state. Seller workflow và Platform workflow ở hai route group của **cùng app**, giữ API namespace và quyền độc lập.

Khi F3 được duyệt, advertiser self-service (nếu có) và Platform moderation quảng cáo/affiliate cùng nằm trong `apps/admin` nhưng dùng capability/route rõ; báo cáo monetization tách mock order value. Finance/Ad Billing UI phải dùng capability khác `ads.manage`, MFA/dual control và data source invoice/receipt/ledger riêng; không thêm real refund vào màn Order mô phỏng. [Advertising](../../domains/advertising.md), [Ad billing](../../domains/advertising-billing.md), [Affiliate](../../domains/affiliate.md).
