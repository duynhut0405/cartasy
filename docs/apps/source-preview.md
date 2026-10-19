---
doc_id: APP-SOURCE-PREVIEW
title: Preview source, thư viện, module và state/store Cartasy
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Bản đồ đọc nhanh của các app/package; chi tiết do app guides, source architecture và stack sở hữu
---

# Preview `src`: dùng gì, đặt ở đâu, lưu state ở đâu

Đây là **blueprint để xem trước khi scaffold**, không phải danh sách package đã cài hay source đang tồn tại. Bốn process F1 là Web, Admin chung Seller/Platform, API và Worker. Thư viện/version lấy từ [stack](../platform/stack-dependencies.md); cây chi tiết, env và gate ở [Web](web/README.md), [Admin](admin/README.md), [API](api/README.md), [Worker](worker/README.md). Boundary package theo [source architecture](../system/source-architecture.md) và [packages](../packages/README.md). Nếu cần đổi module/dep, sửa tài liệu sở hữu trước rồi cập nhật trang preview này.

| Source | Framework và thư viện nổi bật | State/store chính |
|---|---|---|
| `apps/web/src` | Next/React, `next-intl`, React Query, OpenAPI Fetch, React Hook Form/Zod, UI kit | URL + React local + React Query; guest cart `localStorage`; checkout attempt `sessionStorage` |
| `apps/admin/src` | Next/React, React Query, OpenAPI Fetch, React Hook Form/Zod, UI kit | URL + React local + React Query theo actor/workspace/shop; API giữ quyền |
| `apps/api/src` | Nest/Express, Better Auth, DTO validation, Swagger | Request/DI tạm; business state ở PostgreSQL qua `server`/`database` |
| `apps/worker/src` | Nest application context, Nodemailer, Web Push, S3 adapter | PostgreSQL outbox/inbox/jobs; object storage cho media |
| `packages/*/src` | Pure TypeScript domain, Drizzle/pg, generated contract, React UI, Zod config, Pino | PostgreSQL là nguồn nghiệp vụ; package chỉ giữ contract/adapter/token |

## Toàn cảnh dependency và dữ liệu

```text
apps/web ──────┐
               ├── @cartasy/contracts + @cartasy/ui + public config
apps/admin ────┘                  │
       │ typed HTTP + cookie      │
       └───────────> apps/api ────┼──> @cartasy/server (rule/use case/port)
                         │        └──> @cartasy/database (Drizzle/pg adapter)
                         │                       │
                         │                       └──> PostgreSQL
                         └──> outbox/jobs ──> apps/worker ──> email/push/media
```

`@cartasy/server` không biết Next/Nest/Drizzle; `@cartasy/database` triển khai port của server; API/Worker chỉ điều phối adapter. Web/Admin không import server/database. Cartasy order/payment/shipping/refund vẫn mô phỏng. F2/F3 chỉ là namespace dự kiến, không cài package hoặc tạo module rỗng trong F1.

## 1. `apps/web/src` — Storefront Buyer

| Khu vực | Module/thư viện chính | State/store |
|---|---|---|
| `proxy.ts`, `i18n/*`, `app/[locale]/*`, `messages/{vi,en}` | Next.js App Router, React, `next-intl`; `/vi` và `/en`, metadata/sitemap | Locale và search/filter/sort/cursor ở URL; public Server Component fetch/cache có key theo locale |
| `widgets/*`, `entities/*`, `shared/ui/*` | `@cartasy/ui`, Tailwind CSS, Radix primitive cần dùng, `lucide-react` | Props từ API; tab/dialog/selection tạm trong React component |
| `features/discovery`, `product-detail`, `wishlist`, `order-history`, `reviews`, `returns`, `support`, `account`, `notifications` | React, `@tanstack/react-query`, `openapi-fetch`, `@cartasy/contracts` | React Query cache cho response/mutation; query key có actor/resource/filter/locale khi chứa text |
| `features/cart`, `checkout`, `payment-result` | React Query, `react-hook-form`, `zod`, `@hookform/resolvers` | Guest cart IDs/qty trong `localStorage` versioned; cart/order Buyer ở API/PostgreSQL; form local; opaque checkout idempotency key trong `sessionStorage` theo **một attempt** |
| `shared/api`, `auth`, `realtime`, `config`, `styles`, `public/sw.js` | Typed API client, browser `EventSource`, Push/Notification API, VAPID public key | Session cookie `HttpOnly` do API quản lý; SSE chỉ báo refetch; Web Push subscription do API lưu |

Luồng UI: `app` compose `widgets/features`; `features` dùng `entities/shared`; chỉ gọi public API của feature khác qua composition. Giá, tồn kho, discount, order/payment state do server quyết định. `next-intl` JSON chỉ chứa **UI copy**; catalog/shop/CMS translations nằm trong DB theo owner. [Cấu trúc đầy đủ](web/README.md).

## 2. `apps/admin/src` — một workspace Seller và Platform

| Khu vực | Module/thư viện chính | State/store |
|---|---|---|
| `app/(auth)`, `(workspace)`, `(seller)/seller/*`, `(platform)/platform/*` | Next.js App Router/React; một source tree, một origin | URL giữ workspace/shop/resource/filter/cursor; quyền được kiểm lại ở API |
| `widgets/workspace-shell`, `seller-shell`, `platform-shell`, `filter-toolbar`, `audit-diff`, `translation-editor` | `@cartasy/ui`, Tailwind, Radix, icon | Shell/menu theo context từ `/v1/portal/context`; không là bằng chứng cấp quyền |
| Seller features: onboarding, shop switch/profile/membership, product proposal, listing/inventory/order, promotion, notification proposal, support/return/review, reporting | React Query, `openapi-fetch`, generated contracts; React Hook Form/Zod cho form | Query key có actor + workspace + shop ID + membership/capability version; đổi shop hủy request/stream và clear private cache/form |
| Platform features: catalog/listing/inventory/promotion/order/return/review/risk/support, shop governance, CMS/campaign, reporting, access, operations | Cùng typed client/form/UI kit; API `/v1/admin/*` theo capability | Draft form local, `If-Match`/version cho mutation; server lưu dữ liệu và audit; không optimistic update cho action nhạy cảm |
| `shared/api`, `auth`, `realtime`, `config`, `ui`, `styles` | React Query, `EventSource`, API cookie/session | Session và capability do API quyết định; không lưu authority trong localStorage |

Admin hiện **không có app Seller riêng**. Bản dịch catalog/shop/CMS được biên tập tại đây; route Admin `/seller/*`/`/platform/*` không cần locale prefix SEO. Dịch UI Admin bằng `next-intl` là khả năng mở rộng trong app hiện có, chưa là dependency bắt buộc của F1. [Cấu trúc đầy đủ](admin/README.md).

## 3. `apps/api/src` — HTTP boundary

| Khu vực | Module/thư viện chính | State/store |
|---|---|---|
| `bootstrap/*`, `composition/*` | NestJS 12/Express, `reflect-metadata`, `rxjs`; bind server use case + database port, sinh OpenAPI | Nest DI/application context; không giữ business state trong singleton process |
| `platform/auth/*` | Better Auth, session/capability/shop/CSRF guards | Session Better Auth/cookie API origin; RBAC + shop capability + ABAC ở application layer |
| `platform/http/*` | `class-validator`, `class-transformer`, request ID, error filter, locale DTO, idempotency/ETag | Request/DTO tạm; idempotency record và version thật ở PostgreSQL |
| `modules/*` | Controller, DTO, mapper, Nest module; `@nestjs/swagger` + `@cartasy/contracts` | Controller không sở hữu nghiệp vụ/SQL; query/command gọi `@cartasy/server` |

F1 HTTP modules: `identity`, `customers`, `communications`, `notification-campaigns`, `shops`, `seller-onboarding`, `catalog`, `discovery`, `wishlist`, `carts`, `checkout`, `payments`, `orders`, `fulfillment`, `inventory`, `promotions`, `reviews`, `risk`, `returns`, `support`, `media`, `analytics`, `reporting`, `cms`, `jobs`. Một module HTTP có thể compose nhiều owner use case; không suy ra business ownership từ tên controller. `better-auth` mount trước body parser. Public catalog/CMS read có query `locale=vi|en`; auth/private data luôn kiểm quyền. [Cấu trúc đầy đủ](api/README.md), [API contract](../system/api-contracts.md).

## 4. `apps/worker/src` — outbox và job

| Khu vực | Module/thư viện chính | State/store |
|---|---|---|
| `bootstrap/*`, `outbox/*` | Nest application context, `@cartasy/server`/`database`/`config`, `pino` | PostgreSQL `outbox_events` + `inbox_events`; poll/lease/dedupe, không process-memory queue |
| `handlers/payments`, `fulfillment` | Mock settlement/expiry và shipment timeline qua owner use case | Payment/order/stock state ở PostgreSQL; handler idempotent |
| `handlers/notifications` | `nodemailer`, `web-push` qua adapter; recipient/locale/template resolver, campaign batch | Notification/delivery/subscription/consent trong DB; retry giữ template locale/version của recipient |
| `handlers/media`, `discovery`, `analytics` | S3-compatible media adapter, search projection, event projection | Object binary ở private object storage; metadata/projection/checkpoint ở PostgreSQL |
| `jobs/*`, `scheduler/*`, `health/*` | Poller, lease/checkpoint, fake clock test, metrics/log | PostgreSQL background jobs + advisory/distributed lock; dead-letter và lag có metric |

Worker không có HTTP public. SMTP/Push/S3 gọi ngoài transaction; event xử lý at-least-once nên handler phải idempotent. F1 dùng PostgreSQL polling, không Redis/BullMQ. [Cấu trúc đầy đủ](worker/README.md).

## 5. `packages/*/src` — shared code

| Package | Module/đường dẫn | Thư viện và nguồn state |
|---|---|---|
| `@cartasy/server` | `modules/<context>/{domain,application,ports,public.ts}`, `platform/{clock,ids,transaction,errors}` | TypeScript thuần cho rule/use case/authorization; entity trong một operation, lưu bền qua port chứ không qua singleton store |
| `@cartasy/database` | `client`, `schema`, `repositories`, `read-models`, `migrations`, `seed` | `drizzle-orm`/`pg`, `drizzle-kit` dev; PostgreSQL là nguồn giao dịch, search F1 dùng PostgreSQL full-text/trigram |
| `@cartasy/contracts` | `openapi`, `generated`, `client`, `events` | `openapi-typescript` dev sinh type từ API; wire schema, không có state |
| `@cartasy/ui` | `tokens`, `primitives`, `patterns` | React/Radix/Tailwind token và accessible primitive; nhận translated copy qua props, không gọi API |
| `@cartasy/config` | `env/{api,worker,web,admin}`, `presets` | Zod parse env, public/server allowlist; không là runtime state store |
| `@cartasy/observability` | `logging`, `metrics`, `tracing` | Pino/redaction và correlation; metric/log không là nguồn nghiệp vụ |
| `@cartasy/testkit` | `builders`, `clock`, `fixtures` | Fixture/fake clock chỉ cho test, không đưa runtime image |

`@cartasy/server` F1 có các owner module: identity/customers/shops/seller-onboarding; catalog/media/listings/discovery; inventory/pricing/promotions/carts/wishlist/checkout/payments/orders/fulfillment; notifications/communications; reviews/risk/returns/support; audit/outbox/analytics/cms/reporting/jobs. F2 `flash-sales`, `recommendations`; F3 `advertising`, `affiliate`, `ad-billing` chỉ tạo khi được duyệt và triển khai. [Package guide](../packages/README.md), [source architecture](../system/source-architecture.md).

## State/store chọn cho từng loại dữ liệu

| Dữ liệu | Nguồn chuẩn | Bản sao phục vụ UI/xử lý |
|---|---|---|
| Locale, search/filter/sort/cursor, workspace route | URL `Web /[locale]` hoặc Admin route | Next router; query key có locale/filter thích hợp |
| Tab/dialog/variant đang chọn, input form | React component / React Hook Form | Mất khi rời luồng; không là giá hoặc quyền chính thức |
| Public/private response từ API | Owner use case + PostgreSQL | Server Component public cache hoặc React Query; private cache xóa khi logout/đổi actor/shop |
| Guest cart | Browser `localStorage` adapter versioned, chỉ listing ID/quantity | Merge idempotent vào server cart khi login; không là nguồn giá/stock/PII |
| Checkout attempt key | `sessionStorage` theo actor/cart attempt; PostgreSQL idempotency record là nguồn xử lý | Giữ opaque key qua refresh/timeout, lookup kết quả trước khi clear; không lưu quote/payment detail ở browser |
| Quote/giá hiện tại | `packages/server` tính từ listing/stock/promotion/policy trong PostgreSQL | Response/fingerprint tạm ở UI để xác nhận; checkout tính và kiểm lại, không lấy tổng từ client |
| Buyer cart, order, payment, stock, consent, quyền | `packages/server` rule + PostgreSQL qua `database` | UI chỉ đọc/gửi command; không có Redux/Zustand làm nguồn chính thức |
| Session | Better Auth ở API + `HttpOnly` cookie | Web/Admin gọi API với `credentials: include` |
| Event, job, retry, delivery | PostgreSQL outbox/inbox/background jobs | Worker poll/lease/dedupe; SSE chỉ hint cho client refetch |
| Media binary | Private S3-compatible object storage | PostgreSQL giữ key/metadata/quyền; signed URL ngắn hạn |

MVP không chọn Redux/Zustand, Redis/BullMQ, WebSocket server, FCM hay search engine ngoài PostgreSQL. Nếu phát sinh nhu cầu sau đo tải, phải có ADR/contract/gate tương ứng; không thêm một store thứ hai chỉ để tránh truyền props hoặc dùng React Query.

## Tooling và test gắn với source

| Source | Dev/test dependency và gate chính |
|---|---|
| Root workspace | `pnpm`, `turbo`, TypeScript strict, ESLint/Prettier, `dependency-cruiser`, `knip`, `secretlint`; frozen lockfile và boundary check |
| Web/Admin | `eslint-config-next`, Vitest/component test khi cần, `@playwright/test` ở `tests/e2e` cho hai locale, checkout và cross-role/shop; build độc lập |
| API | `@nestjs/testing`, `supertest`, Vitest, PostgreSQL integration, OpenAPI/`openapi-typescript` drift và auth route snapshot |
| Worker | Vitest + PostgreSQL integration/fake clock: event trùng, lease/retry/dead-letter, mock settlement và delivery |
| Database/contracts/UI | Migration clean+upgrade, generated type check, UI accessibility/interaction và package export/boundary check |

Chỉ đánh dấu gate hoàn thành khi có lệnh chạy và artifact theo [Testing](../platform/testing-quality.md) và [Build checklist](../delivery/build-checklist.md). Phiên bản exact/peer compatibility được pin khi scaffold theo [stack](../platform/stack-dependencies.md).
