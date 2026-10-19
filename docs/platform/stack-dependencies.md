---
doc_id: TECH-STACK
title: Cartasy Tech Stack and Dependencies
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Toolchain, thư viện bắt buộc, version policy và compatibility gate
review_cycle: Mỗi lần nâng major hoặc đổi runtime
---

# Tech stack và dependency policy

## 1. Version policy

Baseline kiểm tra ngày 19/10/2026: Node.js `24.22.0` LTS, pnpm `12.11.2`, TypeScript `6.0.3`, Next.js `16.4.2`/React `19.3.1`, NestJS `12.1.3`, Better Auth `1.7.11`, Drizzle ORM `0.45.4`/Kit `0.31.11` và PostgreSQL `18.x`. Dùng một `.node-version`, `packageManager` exact trong root manifest và một `pnpm-lock.yaml`. Đây là điểm khởi tạo, không phải tuyên bố các package đã được cài; khi scaffold phải kiểm tra peer dependency, pin exact patch tương thích trong manifest/lockfile và chạy đầy đủ gate. Không dùng `latest` trong runtime/deploy. Nâng major bằng ADR và migration plan, không trộn với feature. TypeScript 6 được chọn vì `@nestjs/swagger` 12 công bố peer range tới 6; chỉ nâng TypeScript 7 khi chuỗi Nest/Swagger và test đã hỗ trợ.

NestJS 12/Better Auth Express chạy ESM. `apps/api` phải mount Better Auth handler trước body parser theo [hướng dẫn Express chính thức](https://better-auth.com/docs/integrations/express); kiểm thử route auth khi thay Express/Nest major. Next.js dùng App Router trên Node server/container để giữ đầy đủ tính năng ([Next.js deployment](https://nextjs.org/docs/app/getting-started/deploying)). Không đặt Web/Admin trên edge adapter trong baseline; Seller dùng cùng Admin app theo ADR-0011.

## 2. Danh sách thư viện

| Lớp | Package | Mục đích/quy tắc |
|---|---|---|
| Workspace | `pnpm`, `turbo` | Workspace và task graph |
| Ngôn ngữ | `typescript`, `@types/node` | Strict ESM, typecheck độc lập |
| Web/Admin chung Seller-Platform | `next`, `react`, `react-dom` | SSR, App Router, SEO và UI |
| Web i18n | `next-intl` | `/[locale]`, UI messages, navigation/formatting; pin exact patch tương thích Next 16 khi scaffold, kiểm peer/build cả hai locale |
| UI | `tailwindcss`, `@tailwindcss/postcss`, `@radix-ui/react-dialog`, `@radix-ui/react-alert-dialog`, `@radix-ui/react-checkbox`, `@radix-ui/react-radio-group`, `@radix-ui/react-select`, `@radix-ui/react-tabs`, `@radix-ui/react-label`, `@radix-ui/react-slot`, `lucide-react` | Token/primitive accessible; chỉ cài package được component thực tế dùng |
| Client data | `@tanstack/react-query`, `openapi-fetch` | Cache server state ở Client Component, typed HTTP client |
| Form | `react-hook-form`, `@hookform/resolvers`, `zod` | Form state và client validation; server vẫn validate độc lập |
| API | `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express`, `@nestjs/swagger`, `reflect-metadata`, `rxjs` | HTTP, module DI và OpenAPI |
| API validation | `class-validator`, `class-transformer` | DTO boundary, allowlist/whitelist; không dùng DTO làm domain model |
| Auth | `better-auth` | Email/password, verification, session và MFA admin/seller |
| Database | `drizzle-orm`, `drizzle-kit`, `pg` | SQL schema, migration và PostgreSQL client |
| Contract | `openapi-typescript` | Generate type từ OpenAPI vào `packages/contracts`; cấm sửa artifact bằng tay |
| Object storage | `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner` | Media adapter; bucket private |
| Email | `nodemailer` | SMTP adapter; Mailpit local, provider production qua cấu hình |
| Web Push | `web-push@3.6.7`, `@types/web-push@3.6.4` (dev) | VAPID sender trong Worker, service worker dùng browser Push/Notification API; kiểm ESM interop/Node 24 khi scaffold |
| Realtime | NestJS `@Sse()`/`rxjs`, browser `EventSource` | SSE one-way hint cho inbox/order; không thêm WebSocket package trong MVP |
| Logging | `pino` | Structured log có redaction; OpenTelemetry SDK chỉ thêm khi trace sink được chọn |
| Unit/integration | `vitest`, `@nestjs/testing`, `supertest` | Domain, HTTP và PostgreSQL integration |
| Browser | `@playwright/test` | E2E Web và Admin với Seller/Platform role |
| Quality | `eslint`, `typescript-eslint`, `eslint-config-next`, `prettier`, `prettier-plugin-tailwindcss`, `dependency-cruiser`, `knip` | Lint/format, dependency boundary, unused export |
| Security | `secretlint`, `pnpm audit` | Secret/dependency scan trong CI |

Dev type packages theo consumer: `@types/react`, `@types/react-dom`, `@types/pg`, `@types/nodemailer` và `@types/supertest`. Chỉ khai báo dependency trực tiếp ở workspace dùng nó; `drizzle-kit`, `openapi-typescript`, test/lint/formatter là dev dependency, không đưa vào runtime image.

Không dùng Redis/BullMQ, search engine, FCM, WebSocket server, payment SDK thật, carrier SDK hoặc state library toàn cục trong **F1 baseline**. Ad billing provider SDK/partner affiliate API sau MVP là quyết định riêng theo hợp đồng và [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md); không thêm vào runtime F1 hoặc mock checkout. PostgreSQL outbox polling, React Query và URL state đáp ứng F1. Nếu dùng component generator, commit code được sinh và review như code thường; không coi generator là runtime dependency.

Các lựa chọn dựa trên khả năng được tài liệu chính thức mô tả: [NestJS modules](https://docs.nestjs.com/first-steps), [Drizzle migrations](https://orm.drizzle.team/docs/migrations), [NestJS OpenAPI](https://docs.nestjs.com/openapi/introduction), [OpenAPI Fetch](https://openapi-ts.dev/openapi-fetch/), [Better Auth](https://better-auth.com/docs/installation) và [pnpm 12](https://pnpm.io/installation).

## Hướng dẫn tiếp theo

[Local development](local-development.md) sở hữu dịch vụ local, cổng, root manifest, biến môi trường và thứ tự bootstrap. [App guides](../apps/README.md) sở hữu dependency trực tiếp, cây `src` và setup của từng ứng dụng. [Migrations](database-migrations.md) sở hữu quy trình schema/seed. Không sao chép các lệnh setup ở đây. Web Push dùng chuẩn browser/VAPID; FCM/APNs chỉ khi có app mobile native hoặc ADR cụ thể. [Integrations](external-integrations.md) sở hữu transport và webhook.
