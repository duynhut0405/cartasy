---
doc_id: APP-API
title: Backend/API
status: accepted
owner: Backend Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Setup, src, HTTP boundary, DI và integration của apps/api
---

# Backend/API

[Preview thư viện, module và state/store của toàn workspace](../source-preview.md).

## Vai trò và stack

`apps/api` là NestJS 12 ESM trên Node 24/Express adapter, composition root của `@cartasy/server` use cases và `@cartasy/database` adapters. Trực tiếp dùng `@nestjs/common/core/platform-express/swagger`, `reflect-metadata`, `rxjs`, `class-validator`/`class-transformer` ở HTTP DTO, `better-auth`, `zod` cho env, `pino` cho log. Drizzle/pg chỉ trong database package; SDK SMTP/S3 chỉ adapter đúng owner. OpenAPI sinh từ controller/DTO, `openapi-typescript` tạo `@cartasy/contracts`. Lựa chọn/version chuẩn ở [stack](../../platform/stack-dependencies.md).

## `src` và chiều phụ thuộc

```text
src/
  bootstrap/
    main.ts
    app.module.ts
    configure-http.ts
    configure-auth.ts
    openapi.ts
  platform/
    auth/
      session.guard.ts
      capability.guard.ts
      shop-policy.guard.ts
      csrf-origin.guard.ts
    http/
      request-id.middleware.ts
      exception.filter.ts
      validation.pipe.ts
      idempotency.interceptor.ts
      pagination.dto.ts
      locale-query.dto.ts           # vi/en enum cho public read, không là auth
    config/
    health/
  modules/
    identity/
    customers/
    communications/
    notification-campaigns/
    shops/
    seller-onboarding/
    catalog/
    discovery/
    wishlist/
    carts/
    checkout/
    payments/
    orders/
    fulfillment/
    inventory/
    promotions/
    reviews/
    risk/
    returns/
    support/
    media/
    analytics/
    reporting/
    cms/
    jobs/
  composition/
    providers.ts
    transaction.ts
```

Trong `modules/<name>`: `<name>.controller.ts`, `dto/` request/response, `<name>.mapper.ts`, `<name>.module.ts`; không viết domain rule/SQL trong controller. Business logic nằm trong `packages/server/src/modules/<name>/{domain,application,ports,public.ts}`; Drizzle schema/repository adapter trong `packages/database`. App import public exports; database adapter implements ports; không import vòng. Những module flash sale/recommendation/ads/affiliate F2/F3 chỉ tạo khi triển khai theo [source architecture](../../system/source-architecture.md).

## HTTP, auth và contract

Mount Better Auth handler `/v1/auth/*` **trước body parser** theo integration Express; test route provider sau mỗi upgrade. Auth session cookie host-only API origin, SameSite Lax/Secure HTTPS; allowlist Web/Admin origin, credentialed CORS, Origin/CSRF check cho mọi mutation. Default guard từ chối trừ `@Public` được khai báo; role, shop capability và resource attributes kiểm tra ở application layer, không chỉ guard path. Admin/seller MFA. `GET /v1/portal/context` trả role/capability/membership tối thiểu cho một Admin workspace, không cấp quyền thay business route. DTO whitelist/forbid unknown, giới hạn body/upload, request ID; error filter map domain error sang envelope và status ở [API contracts](../../system/api-contracts.md). Money string, UTC, cursor, ETag/`If-Match` và `Idempotency-Key` thống nhất. Không nhận giá, actor, shop, payment outcome từ browser.

OpenAPI artifact `packages/contracts/openapi/cartasy.v1.json` và generated TS types là output từ controller/DTO; CI fail khi drift. Provider auth route cần snapshot contract riêng. Breaking change đi qua `/v2` hoặc giai đoạn tương thích N/N-1 và migration. SSE `/v1/me/stream`, seller/admin stream kiểm session/capability và phát hint với cursor/replay, connection cap; API nhận Web Push subscription encrypted at rest, không gửi push trực tiếp. Campaign/notification DTO kiểm audience, consent, relative URL và state; không nhận arbitrary recipient list. Health: `/health/live` không phụ thuộc DB, `/health/ready` kiểm tra DB và dịch vụ bắt buộc; rate limit auth/checkout/support.

## Env và setup độc lập

`APP_ENV`, `PORT=3002`, `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL=http://localhost:3002/v1/auth`, `WEB_URL`, `ADMIN_URL`, `SMTP_HOST/PORT/EMAIL_FROM` theo adapter, `S3_*`, `MOCK_PAYMENT_MODE=enabled`, `LOG_LEVEL`. Parse fail-fast; secret chỉ ở server và secret manager. Không có `SELLER_URL` hoặc CORS wildcard; Seller dùng Admin origin. Danh sách đầy đủ ở [stack](../../platform/stack-dependencies.md).

1. Pin Node/pnpm, cài workspace và bật PostgreSQL/Mailpit/object store theo [local setup](../../platform/local-development.md).
2. Tạo package `@cartasy/api` scripts `dev/build/start/lint/typecheck/test`, Nest ESM TS NodeNext, cây `src` và DI token cho ports. Mount auth đúng thứ tự; tạo health/error/request-id trước business endpoint.
3. Apply migration đã review rồi seed local. Chạy `pnpm --filter @cartasy/api dev`, verify `/health/ready`, auth cookie, public catalog, protected 401, CSRF, SSE và OpenAPI.
4. Sinh contract `pnpm api:generate`, kiểm `pnpm api:check`; chạy API unit/integration trên PostgreSQL thật, test module isolation, shop cross-access, idempotency, transaction rollback.

Quy trình module: domain invariant → use case/ports → schema/repository → controller/DTO/OpenAPI → auth policy/audit/outbox → integration test → typed client. Không gọi external SDK hoặc network trong transaction checkout. HTTP query không làm side effect.

F3 advertiser/affiliate API chỉ mở khi owner domain và partner/provider contract được duyệt. `/vN/affiliate/out/:opaqueCode` là redirect được kiểm allowlist/consent, không là buyer checkout; ad billing callback/receipt thuộc module riêng, không map sang `paid_simulated`. Quy tắc tại [API contracts](../../system/api-contracts.md), [ADR-0010](../../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).
