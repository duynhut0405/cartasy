---
doc_id: PLATFORM-LOCAL
title: Local development và bootstrap workspace
status: accepted
owner: Developer Experience
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Thứ tự setup chung, cổng và lệnh workspace
---

# Local development

Tài liệu này là quy trình **cần tạo** khi khởi tạo source từ bộ docs, không tuyên bố các manifest/Compose đã có. Phiên bản package và cấu hình root mẫu ở [stack](stack-dependencies.md); cây source ở [source architecture](../system/source-architecture.md). Có thể setup app riêng theo [app guide](../apps/README.md) sau khi dependency chung sẵn sàng.

## Điều kiện và dịch vụ

Node 24.22.0, pnpm 12.11.2, Docker/Compose, PostgreSQL 18, Mailpit, S3-compatible object store. Cổng mặc định: Web 3000, Admin chung Seller/Platform 3001, API 3002, PG 5432, Mailpit SMTP/UI 1025/8025, object store API/console 9000/9001. Không có cổng Seller 3003. Nếu đổi cổng, cập nhật đồng thời origin, CORS, env và callback/auth URL. Local Web/Admin/API nên dùng host cùng site (localhost) để cookie API hoạt động; không trộn `localhost` với `127.0.0.1`. HTTPS/same-site sibling domains bắt buộc staging/production.

## Thứ tự bootstrap từ workspace trống

1. Tạo root `package.json` private ESM, `pnpm-workspace.yaml` (`apps/*`/`packages/*`), `.node-version`, `turbo.json`, strict TS base, ESLint, Prettier, dependency-cruiser và `pnpm-lock.yaml`. Pin exact patch/peer-compatible từ [stack](stack-dependencies.md); `pnpm install --frozen-lockfile` sau khi có lock.
2. Tạo `packages/config`, `contracts`, `ui`, `observability`, `testkit`, `server`, `database` và public exports. Tạo `apps/api`, `worker`, `web`, `admin` theo app guides ngay F0; `/seller/*` và `/platform/*` ở cùng `apps/admin`.
3. Tạo `infra/compose` cho PostgreSQL/Mailpit/object store có healthcheck và pinned image; `docker compose up -d`. Tạo `.env.example` chỉ tên biến/giá trị local vô hại; copy thành `.env.local` riêng mỗi app, secret không commit.
4. Tạo DB schema/migrations theo [migrations](database-migrations.md), apply vào DB sạch và seed hai shop độc lập/hai seller, category/product/variant/listing/demo buyer/admin **chỉ local**. Không seed review/order giả như sự kiện thật.
5. Chạy API + Worker rồi Web/Admin; kiểm tra health/readiness, Mailpit, object store. Seed bản `vi` và ít nhất một product/CMS bản `en` đã publish cùng một bản `en` chưa publish. Smoke `/vi`/`/en`, locale switch/404, auth cookie/login/MFA, Seller và Platform workspace, catalog, cart, checkout, payment success/failure/timeout, notifications, order, return/support.
6. Sinh OpenAPI/type; chạy `pnpm check` gồm format/lint/typecheck/unit/integration/contract/migration/build/E2E theo [testing](testing-quality.md). Chỉ báo hoàn thành scaffold khi gate và smoke có bằng chứng.

## Scripts bắt buộc và trách nhiệm

| Script root | Thực hiện |
|---|---|
| `dev`, `dev:web/admin/api/worker` | Chạy từng process hoặc tất cả có log riêng |
| `db:generate`, `db:migrate`, `db:seed`, `db:check` | SQL migration, apply, seed local/test, drift/upgrade check |
| `api:generate`, `api:check` | OpenAPI và generated type, kiểm drift |
| `format`, `format:check`, `lint`, `typecheck` | Static quality/architecture |
| `test`, `test:integration`, `test:e2e` | Domain, PostgreSQL/HTTP, browser |
| `build`, `check` | Build độc lập và gate tổng |

Mỗi workspace có script tương ứng; dùng `pnpm --filter @cartasy/<app> <script>` để phát triển riêng. `pnpm check` không được “pass” bằng cách bỏ test hoặc tắt boundary rule. CI cài bằng frozen lockfile, chạy migration trên DB sạch và fixture upgrade. [Deployment](deployment-operations.md) sở hữu staging/production release.

## Cấu hình root tối thiểu để khởi tạo

```json
{
  "name": "cartasy",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@12.11.2",
  "engines": { "node": ">=24 <25" },
  "scripts": {
    "dev": "turbo run dev --parallel",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test"
  }
}
```

```yaml
packages:
  - apps/*
  - packages/*
```

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "dev": { "cache": false, "persistent": true },
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] },
    "test": {}
  }
}
```

Lưu lần lượt thành `package.json`, `pnpm-workspace.yaml` và `turbo.json`; thêm các script database/API/quality gate ở trên trước khi đánh dấu `F0-01`. Server TS dùng `module/moduleResolution: NodeNext`, `experimentalDecorators` và `emitDecoratorMetadata` cho Nest; Next dùng `moduleResolution: Bundler` theo generator. Mọi workspace bật `strict` và `noUncheckedIndexedAccess`. API/worker xuất ESM thật, không trộn CommonJS output. Cài dependency theo bảng, tạo lockfile, rồi dùng `pnpm install --frozen-lockfile` trong CI. Cài browser E2E một lần bằng `pnpm exec playwright install chromium` và engine thứ hai được chọn cho release.


## Environment contract

| Biến | Process | Ý nghĩa |
|---|---|---|
| `APP_ENV` | Server | `local`/`test`/`staging`/`production` |
| `WEB_URL`, `ADMIN_URL`, `API_URL` | All theo scope | URL public, allowlist origin; Seller dùng `ADMIN_URL` |
| `DATABASE_URL` | API/worker/migration | PostgreSQL; SSL theo môi trường |
| `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL` | API | Secret entropy cao và URL auth |
| `SMTP_HOST`, `SMTP_PORT`, `EMAIL_FROM` | Worker/API | Email adapter |
| `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | API/worker | Object storage; không đưa vào browser |
| `MOCK_PAYMENT_MODE` | API/worker | Chỉ cho phép `enabled` ở baseline, production vẫn mô phỏng |
| `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` | Worker/API theo trách nhiệm | Web Push sender; private key không đưa browser |
| `NEXT_PUBLIC_VAPID_PUBLIC_KEY` | Web | Public subscription key; phải khớp server |
| `WORKER_CONCURRENCY`, `OUTBOX_POLL_MS` | Worker | Bounded polling/job fanout config |
| `LOG_LEVEL` | Server | Structured log |

Client chỉ nhận `NEXT_PUBLIC_API_URL` và URL public của app tương ứng (`NEXT_PUBLIC_SITE_URL` hoặc `NEXT_PUBLIC_ADMIN_URL`) cùng public UI config allowlist. Không có `NEXT_PUBLIC_SELLER_URL`. Không đưa secret, private host hoặc payment control vào `NEXT_PUBLIC_*`. Biến payment scenario override chỉ bật trong test/local theo auth/policy, không cho client production điều khiển kết quả.


## Các ca lỗi setup thường gặp

- CORS/cookie 401 liên tục: kiểm origin chính xác, `credentials: include`, API host-only cookie, SameSite/Secure và không trộn hostname.
- Checkout treo pending: kiểm worker heartbeat/outbox lag, DB lock, mock mode; không gửi lại bằng idempotency key mới.
- Migration drift: đối chiếu schema và SQL migration, không dùng `drizzle-kit push` ở shared database.
- Email không đến: kiểm outbox/delivery và Mailpit; payment/order đã commit không phụ thuộc SMTP.
- Generated client sai: build API/OpenAPI trước, sinh type rồi build Web/Admin.
