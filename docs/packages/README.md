---
doc_id: PACKAGE-GUIDE
title: Shared Packages Architecture
status: accepted
owner: Architecture & Developer Experience
last_updated: 2026-10-20
version: 4.4.1
source_of_truth: Trách nhiệm, src, exports, dependency và setup packages/*
---

# Shared Packages

`packages/*` là hợp đồng dùng chung, **không** là nơi đẩy file của feature bất kỳ. Cây workspace và rule nền ở [Source architecture](../system/source-architecture.md). Không app nào deep-import `@cartasy/*/src/*` hoặc import source của app khác. Chỉ package owner export API có chủ đích qua `package.json#exports`. F0 tạo manifest/skeleton cần thiết; F2/F3 module chỉ tạo khi bắt đầu triển khai.

Khi F3 được duyệt, `advertising`, `affiliate`, `ad-billing` là module owner trong `@cartasy/server` và adapter/schema tương ứng ở database, **không** thêm các kiểu payment thật vào buyer `orders/payments` mô phỏng. Event/DTO namespace và public export tách theo [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

## Package dependency graph

```text
web/admin -> contracts, config(public); shadcn source nằm trong từng app
api/worker -> server, database, contracts(events), config, observability
database -> server/public ports, config(server)
server -> contracts(events only if transport-neutral) / pure primitives
testkit -> server/contracts (dev-only)
```

Không để `server` import HTTP generated client, `database` import app/controller hoặc `contracts` import Drizzle. Domain entity không là HTTP DTO. Dependency-cruiser/ESLint chặn cycle, deep import và server-only dependency trong browser bundle.

## Cấu trúc `src` và trách nhiệm

| Package | Cây chuẩn | Export và quy tắc |
|---|---|---|
| `@cartasy/server` | `src/modules/<context>/{domain,application,ports,public.ts}`, `src/platform/{clock,ids,errors,transaction}` | Pure domain/application use case; một module sở hữu write, module khác gọi public port |
| `@cartasy/database` | `src/{client,schema,repositories,read-models,migrations,seed}` | Drizzle/pg adapter; schema theo owner, một migration stream; không export DB client ra app UI |
| `@cartasy/contracts` | `openapi/cartasy.v1.json`, `src/{generated,client,events}` | OpenAPI/type được sinh, typed client factory, event schema versioned; generated file không sửa tay |
| `@cartasy/config` | `src/env/{api,worker,web,admin}.ts`, `src/presets/` | Zod fail-fast theo process; public env allowlist, không export secret cho browser |
| `@cartasy/observability` | `src/{logging,metrics,tracing}` | Pino/redaction, metric conventions, correlation; provider adapter có thể thay |
| `@cartasy/testkit` | `src/{builders,clock,fixtures}` | Dev dependency, two-shop/two-buyer factories, fake clock và demo data có nhãn |

Web/Admin cài shadcn/ui trực tiếp, mỗi app có `components.json`, source component trong `src/shared/ui` và token/theme trong `src/app/globals.css`. Chỉ add component cần dùng; khi thiếu element hoặc hành vi, compose trong app theo use case. Không tạo package UI F0/F1. [Design system](../system/design-system.md), [ADR-0014](../decisions/ADR-0014-app-owned-shadcn-components.md).

## Mẫu module backend

Ví dụ `notifications`: `domain/notification.ts` giữ trạng thái và invariant; `application/commands/publish-campaign.ts`/`queries/list-inbox.ts` điều phối; `ports/recipient-reader.port.ts`/`delivery-writer.port.ts` mô tả nhu cầu; `public.ts` export command/query/port. DB adapter triển khai port, API/worker bind qua DI. Module `communications` sở hữu channel/template/provider, `notifications` sở hữu inbox/campaign/preference. Queue/outbox là platform adapter, không dùng event emitter trong process làm nguồn delivery.

## Version, build và setup

1. Tạo manifest từng package với `name`, `type: module`, `exports` trỏ source cho dev/built dist cho runtime theo toolchain đã kiểm; `private: true`. Pin direct dependencies tại package consumer. `contracts` có browser-compatible export; UI dependency nằm ở Web/Admin.
2. `pnpm install` từ lockfile; build theo dependency order qua Turbo. ESM NodeNext cho server/database/worker, Bundler resolution cho Next client. TS strict/noUncheckedIndexedAccess.
3. `api:generate` sinh OpenAPI và client; `api:check` fail khi drift. `db:generate` sinh SQL migration để review; production không dùng push schema.
4. Gate theo package: lint/typecheck/build, module unit, database integration với PG thật, API contract/event compatibility và architecture dependency test; accessibility kiểm ở Web/Admin. Testkit không đi vào runtime image.
5. Khi thêm feature: xác định context/data owner → public port/use case → adapter → HTTP/event type → UI. Component thiếu từ shadcn được compose trong app; chỉ xem lại package dùng chung qua ADR khi nhu cầu thực tế ở cả hai app được chứng minh.

[Local development](../platform/local-development.md) sở hữu lệnh root/env; [app guides](../apps/README.md) sở hữu setup từng app.
