---
doc_id: ENG-STANDARDS
title: Cartasy Engineering Standards
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Cấu trúc source code, dependency direction và quy ước phát triển
review_cycle: Khi đổi module, framework hoặc tooling
---

# Quy chuẩn kiến trúc source code

## 1. Nguyên tắc tổ chức

Tổ chức theo **bounded context → module → domain/application/adapter**, không theo route ngẫu nhiên hoặc một thư mục `services` chung. Tầng trong không biết framework hoặc persistence. Một khái niệm có một module sở hữu quyền ghi; phần dùng chung chỉ được trích ra khi là primitive ổn định hoặc contract thực sự được ít nhất hai consumer dùng. Sơ đồ module được thiết kế trước; feature F2/F3 có owner và interface dự kiến nhưng chỉ tạo file khi triển khai.

## 2. Cấu trúc workspace chuẩn

```text
cartasy/
  apps/
    web/                    # Next.js storefront /[locale], next-intl
    admin/                  # Next.js workspace chung Seller + Platform Admin
    api/                    # NestJS HTTP composition root
    worker/                 # outbox/scheduler composition root
  packages/
    server/                 # domain + application theo context
    database/               # Drizzle + repository adapters
    contracts/              # OpenAPI và generated API types
    config/                 # typed config, tsconfig/eslint presets
    observability/          # log/metric/trace adapters
    ui/                     # token và accessible primitive
    testkit/                # fixture/factory cho test
  infra/
    compose/                # dependency local
    docker/                 # Dockerfile theo process
  docs/
  tests/
    e2e/
    performance/
```

Không có app import source của app khác. Public export của package nằm trong `package.json#exports`; cấm import `@cartasy/*/src/*`. `packages/testkit` chỉ là dev dependency.

Web tổ chức route trong `app/[locale]`, UI message trong `apps/web/messages`, i18n routing ở `apps/web/src/i18n`; shared UI token/primitive trong `packages/ui`. Catalog/shop/CMS translations do module owner + database adapter sở hữu, không đặt JSON nội dung kinh doanh trong Web. API nhận locale explicit cho public read, Worker dùng locale/template version của recipient khi giao tiếp. [Web guide](../apps/web/README.md), [ADR-0012](../decisions/ADR-0012-multilingual-storefront.md).

## 3. Cấu trúc `packages/server/src`

```text
src/
  modules/
    identity/
    customers/
    shops/
    catalog/
    media/
    listings/
    inventory/
    carts/
    wishlist/
    discovery/
    pricing/
    promotions/
    checkout/
    payments/
    orders/
    fulfillment/
    notifications/
    communications/
    reviews/
    risk/
    returns/
    support/
    audit/
    outbox/
    analytics/
    seller-onboarding/      # F1
    cms/                    # F1
    reporting/              # F1
    jobs/                   # F1
    flash-sales/            # F2
    recommendations/       # F2
    advertising/            # F3
    affiliate/              # F3
    ad-billing/             # sau MVP, chỉ khi real ad billing được duyệt
  platform/
    clock/
    ids/
    transaction/
    errors/
  index.ts
```

Mỗi module được triển khai theo cùng mẫu, ví dụ `checkout`:

```text
modules/checkout/
  domain/
    checkout-policy.ts          # invariant và value object thuần
    checkout-error.ts
  application/
    commands/place-order.ts     # use case; orchestration qua port
    queries/get-checkout-quote.ts
    dto/checkout-result.ts      # application result, không là HTTP DTO
  ports/
    cart-reader.port.ts
    pricing.port.ts
    stock-reservation.port.ts
    order-writer.port.ts
    idempotency.port.ts
    unit-of-work.port.ts
  public.ts                     # chỉ export use case/port được module khác gọi
```

Module nhỏ vẫn giữ chiều phụ thuộc, có thể gộp file domain hợp lý; không tạo class/folder rỗng. `public.ts` là cổng duy nhất để module khác gọi. Các context F2/F3 có namespace trong blueprint nhưng không tạo route, migration hoặc package placeholder nếu chưa triển khai; khi thêm chúng phải dùng mẫu và direction này.

### Trách nhiệm từng tầng

- `domain`: entity/value object, state machine, invariant, domain event thuần TypeScript. Không Nest, Drizzle, HTTP, env.
- `application`: command/query, authorization theo resource, transaction boundary, orchestration và port. Không đọc `process.env` hoặc gọi SDK trực tiếp.
- `ports`: interface nhỏ theo nhu cầu use case; không trả Drizzle row, Express request hay provider response.
- `public.ts`: export có chủ đích; tránh barrel tự động export toàn module.
- `platform`: clock, ID, UnitOfWork abstraction và lỗi chung thật sự cross-context, không chứa business rule.

`checkout` điều phối các owner thông qua port transaction-aware. Adapter database cung cấp một transaction handle; mỗi owner thực thi thao tác ghi của mình trong handle đó. Không để các module tự mở transaction lồng nhau làm mất atomicity.

## 4. Cấu trúc `packages/database/src`

```text
src/
  client/
    pool.ts
    transaction.ts
  schema/
    identity.ts
    customers.ts
    shops.ts
    catalog.ts
    media.ts
    listings.ts
    inventory.ts
    carts.ts
    pricing.ts
    promotions.ts
    flash-sales.ts             # F2 khi triển khai
    orders.ts
    payments.ts
    fulfillment.ts
    notifications.ts
    notification-campaigns.ts
    communications.ts
    push-subscriptions.ts
    cms.ts
    reporting.ts
    jobs.ts
    wishlist.ts
    reviews.ts
    risk.ts
    returns.ts
    support.ts
    analytics.ts
    ad-billing.ts             # chỉ tạo cùng F3 billing, tách payments mô phỏng
    affiliate.ts              # chỉ tạo cùng F3 outbound partner
    advertising.ts            # chỉ tạo cùng F3 ad serving
    idempotency.ts
    audit-outbox.ts
    relations.ts
  repositories/
    identity/
    shops/
    catalog/
    media/
    listings/
    inventory/
    carts/
    wishlist/
    discovery/
    pricing/
    promotions/
    checkout/
    orders/
    payments/
    fulfillment/
    notifications/
    communications/
    cms/
    reporting/
    jobs/
    reviews/
    risk/
    returns/
    support/
    analytics/
    audit/
    outbox/
  read-models/
    public-catalog.ts
    order-history.ts
    admin-orders.ts
    seller-orders.ts
    notification-inbox.ts
    shop-reports.ts
  migrations/
  seed/
  index.ts
```

Schema chia theo owner, migration là một stream chung. Repository implementation map database row sang domain model; read model có thể join nhiều bảng nhưng read-only. Không export database client cho frontend hoặc controller. Seed production chỉ reference data; demo data chỉ local/test.

## 5. Cấu trúc các package dùng chung

```text
packages/
  contracts/
    openapi/cartasy.v1.json       # artifact sinh từ API
    src/generated/api-v1.ts       # openapi-typescript, không sửa tay
    src/client/public.ts          # typed client factory, không chứa cookie/URL cố định
    src/events/                   # schema versioned cho outbox/inbox
  config/
    src/env/{api,worker,web,admin}.ts # parser theo process, public env allowlist
    src/presets/                  # tsconfig/eslint/build preset
  ui/
    src/tokens/                   # màu, typography, spacing
    src/primitives/               # accessible input/dialog/button...
    src/patterns/                 # composition thật sự dùng nhiều app
  observability/
    src/logging/                  # Pino/redaction/context
    src/metrics/                  # metric names và exporter adapter
    src/tracing/                  # correlation/trace adapter nếu có sink
  testkit/
    src/builders/                 # domain fixture builder
    src/clock/                    # fake clock
    src/fixtures/                 # dữ liệu mô phỏng có nhãn
```

`contracts` chỉ có wire schema/type, không import domain/database. `ui` không gọi API hoặc chứa chính sách order/price; app compose primitive thành feature. `config` chỉ parse/validate env, không xuất server secret vào browser. `observability` không ghi PII trong label/log. `testkit` chỉ là dev dependency, không đóng gói runtime. Chỉ trích xuất vào package chung khi ít nhất hai consumer thực sự cần cùng abstraction ổn định; mỗi package có public `exports`, không deep import.

## 6. Ranh giới source của ứng dụng

Cây `src`, route, state, API integration, biến môi trường và lệnh setup chi tiết chỉ do tài liệu ứng dụng sở hữu: [Web](../apps/web/README.md), [Admin chung Seller/Platform](../apps/admin/README.md), [API](../apps/api/README.md), [Worker](../apps/worker/README.md). Không sao chép lại cây app ở đây để tránh lệch khi triển khai.

- `apps/web`, `apps/admin` là presentation adapter; chỉ import `contracts`, `ui`, config public và primitive phù hợp. `app` compose route; `features` thao tác; `entities` model trình bày; `widgets` tổ hợp; `shared` không biết domain.
- `apps/api` là HTTP composition root Nest; controller/DTO/mapper/guard chỉ chuyển đổi boundary, không chứa SQL/state transition/tính tiền. Better Auth mount trước body parser.
- `apps/worker` là background composition root; handler/scheduler gọi cùng use case ở `packages/server`, sử dụng outbox/inbox dedupe và không tạo biến thể rule.
- `apps/admin` chứa route group `/seller/*` và `/platform/*` trong cùng source tree; không dùng chung authorization client cache key giữa hai workspace. API tiếp tục tách namespace seller/admin và kiểm RBAC + shop membership + thuộc tính tài nguyên ở application layer theo [ADR-0011](../decisions/ADR-0011-unified-admin-seller-workspace.md).

## 7. Quy tắc code và phụ thuộc

- TypeScript `strict`, `noUncheckedIndexedAccess`; dữ liệu ngoài bắt đầu là `unknown` và validate tại boundary. Không dùng `any`/assertion để che lỗi.
- File/folder `kebab-case`; symbol `camelCase`/`PascalCase`; ID có ngữ cảnh (`shopId`, `orderId`); số có đơn vị (`amountMinor`, `timeoutMs`).
- Named export mặc định, default chỉ cho Next route convention. Không tạo `utils.ts` tổng hợp hoặc `common.service.ts`.
- Client không import server/database; server không import adapter; database chỉ import public port của server; package không cycle.
- Field server-owned gồm `user_id`, `shop_id`, price, discount, stock, order/payment status; không mass assign từ DTO.
- SQL query parameterized; quan hệ/hot query có index và query plan. Domain không dùng floating point cho tiền.
- Command thay đổi state; query không có side effect. Network call không ở trong transaction checkout.
- Mọi public API, migration, event schema, UI state và tài liệu liên quan cập nhật cùng feature.
- Prettier là formatter duy nhất; ESLint và dependency-cruiser chặn layer violation/cycle. Không tắt rule ở phạm vi toàn project để sửa một lỗi.

## 8. Cách thêm capability

1. Xác định context owner, invariant, actor và dữ liệu trong tài liệu.
2. Thêm/đổi domain/application/port và test rule.
3. Thêm schema/migration/repository thuộc owner, test PostgreSQL thật.
4. Thêm HTTP/job adapter, DTO/OpenAPI và typed client.
5. Thêm UI theo feature/entity/shared và trạng thái loading/error/permission.
6. Thêm audit/outbox/metric và E2E luồng trọng yếu.
7. Chạy architecture/contract/migration/security gate và cập nhật checklist.

Nếu capability không khớp module hiện có, xem lại bounded context trước khi thêm module; không đặt file vào thư mục gần nhất chỉ vì tiện.
