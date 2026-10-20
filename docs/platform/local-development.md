---
doc_id: PLATFORM-LOCAL
title: Local development và bootstrap workspace
status: accepted
owner: Developer Experience
last_updated: 2026-10-20
version: 4.4.1
source_of_truth: Thứ tự setup chung, cổng và lệnh workspace
---

# Local development

Tài liệu này là quy trình **cần tạo** khi khởi tạo source từ bộ docs, không tuyên bố các manifest/Compose đã có. Phiên bản package và cấu hình root mẫu ở [stack](stack-dependencies.md); cây source ở [source architecture](../system/source-architecture.md). Có thể setup app riêng theo [app guide](../apps/README.md) sau khi dependency chung sẵn sàng.

## Điều kiện và dịch vụ

Node 24.22.0, pnpm 12.11.2, Docker/Compose, PostgreSQL 18, Mailpit, S3-compatible object store. Cổng mặc định: Web 3000, Admin chung Seller/Platform 3001, API 3002, PG 5432, Mailpit SMTP/UI 1025/8025, object store API/console 9000/9001. Không có cổng Seller 3003. Nếu đổi cổng, cập nhật đồng thời origin, CORS, env và callback/auth URL. Local Web/Admin/API nên dùng host cùng site (localhost) để cookie API hoạt động; không trộn `localhost` với `127.0.0.1`. HTTPS/same-site sibling domains bắt buộc staging/production.

## Làm việc trên nhiều máy

Docker image chứa phần mềm, **không chứa dữ liệu PostgreSQL/object store đang ghi vào volume**. Commit Compose/Dockerfile, image reference đã pin, lockfile, SQL migration và seed; máy khác pull/build image từ cùng commit. Compose dùng named volume cho PostgreSQL và object store: dữ liệu còn sau `docker compose down` trên **máy đó**, nhưng `down --volumes` xóa volume và volume không tự đồng bộ sang máy khác. [Docker storage](https://docs.docker.com/engine/storage), [Compose down](https://docs.docker.com/reference/cli/docker/compose/down/).

- **Máy mới, dữ liệu demo chuẩn:** clone/pull cùng commit → tạo env local từ `.env.example` → `pnpm install --frozen-lockfile` → `docker compose pull` và `docker compose up -d` → `pnpm db:migrate` → `pnpm db:seed` → chạy API/Worker/Web/Admin. Seed phải idempotent và có media fixture hợp lệ để hai máy dựng được cùng baseline. Đây là quy trình F0 cần hiện thực; repository hiện chưa có Compose/scripts để chạy ngay.
- **Cần tiếp tục đúng trạng thái dữ liệu giữa các máy:** dùng **một PostgreSQL dev riêng cho người phát triển** và bucket dev tương ứng, truy cập từ cả hai máy bằng `DATABASE_URL`/S3 env riêng từng máy qua TLS và mạng giới hạn. Không dùng staging/production hoặc DB test/CI. Chỉ máy đang làm việc chạy Worker; dùng cùng commit/schema, chạy migration có kiểm soát một lần và backup trước thay đổi phá dữ liệu. Khởi tạo seed một lần trên DB rỗng, không tự reset/seed lại mỗi khi mở máy.
- **Muốn giữ local và chuyển trạng thái khi cần:** tạm dừng API/Worker ghi dữ liệu, xuất `pg_dump` custom format, khôi phục bằng `pg_restore` vào DB rỗng ở máy đích; chuyển cả object store nếu dữ liệu tham chiếu media upload. Sau restore chạy `db:check` và migration còn thiếu. Mã hóa snapshot, để ngoài Git; không sao chép dữ liệu thật/PII từ production. [PostgreSQL backup/restore](https://www.postgresql.org/docs/current/backup.html).

Integration test vẫn dùng database cô lập trên từng máy. Không dùng Docker image hay copy trực tiếp thư mục volume làm cơ chế đồng bộ dữ liệu phát triển.

## Thứ tự bootstrap từ workspace trống

1. Tạo root `package.json` private ESM, `pnpm-workspace.yaml` (`apps/*`/`packages/*`), `.node-version`, `turbo.json`, strict TS base, ESLint, Prettier + Tailwind plugin, Husky/lint-staged, dependency-cruiser và `pnpm-lock.yaml`. Pin exact patch/peer-compatible từ [stack](stack-dependencies.md); `pnpm install --frozen-lockfile` sau khi có lock.
2. Tạo `packages/config`, `contracts`, `observability`, `testkit`, `server`, `database` và public exports. Tạo `apps/api`, `worker`, `web`, `admin` theo app guides ngay F0; `/seller/*` và `/platform/*` ở cùng `apps/admin`. Cấu hình `components.json` riêng trong `apps/web` và `apps/admin`, aliases trỏ component shadcn vào `src/shared/ui` của chính app; pin shadcn CLI, chọn bản Radix rõ ràng khi init, commit component source đã sinh. Chỉ thêm component theo màn hình và compose component riêng khi shadcn thiếu. CSS app chỉ là Tailwind/theme entry, feature/page dùng utilities.
3. Tạo `infra/compose` cho PostgreSQL/Mailpit/object store có healthcheck, pinned image và named volume cho dữ liệu; `docker compose up -d`. Tạo `.env.example` chỉ tên biến/giá trị local vô hại; copy thành `.env.local` riêng mỗi app, secret và snapshot không commit. Hỗ trợ cấu hình chỉ chạy Mailpit khi API/Worker dùng DB và bucket dev từ xa.
4. Tạo DB schema/migrations theo [migrations](database-migrations.md), apply vào DB sạch và seed hai shop độc lập/hai seller, category/product/variant/listing/demo buyer/admin **chỉ local**. Không seed review/order giả như sự kiện thật.
5. Chạy API + Worker rồi Web/Admin; kiểm tra health/readiness, Mailpit, object store. Seed bản `vi` và ít nhất một product/CMS bản `en` đã publish cùng một bản `en` chưa publish. Smoke `/vi`/`/en`, locale switch/404, auth cookie/login/MFA, Seller và Platform workspace, catalog, cart, checkout, payment success/failure/timeout, notifications, order, return/support.
6. Sinh OpenAPI/type; chạy `pnpm check` gồm format/lint/typecheck/unit/integration/contract/migration/build/E2E theo [testing](testing-quality.md). Chỉ báo hoàn thành scaffold khi gate và smoke có bằng chứng.

## Scripts bắt buộc và trách nhiệm

| Script root | Thực hiện |
|---|---|
| `dev`, `dev:web/admin/api/worker` | Chạy từng process hoặc tất cả có log riêng |
| `db:generate`, `db:migrate`, `db:seed`, `db:check` | SQL migration, apply, seed local/test, drift/upgrade check |
| `api:generate`, `api:check` | OpenAPI và generated type, kiểm drift |
| `format`, `format:check`, `lint`, `typecheck` | Prettier format/check source, Tailwind class order, ESLint và architecture boundary |
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
    "prepare": "node .husky/install.mjs",
    "format": "prettier --write \"apps/**/src/**/*.{ts,tsx,js,jsx,json,css,md}\" \"packages/**/src/**/*.{ts,tsx,js,jsx,json,css,md}\"",
    "format:check": "prettier --check \"apps/**/src/**/*.{ts,tsx,js,jsx,json,css,md}\" \"packages/**/src/**/*.{ts,tsx,js,jsx,json,css,md}\"",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test"
  },
  "lint-staged": {
    "**/src/**/*.{ts,tsx,js,jsx,json,css,md}": "prettier --write --ignore-unknown"
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

Lưu lần lượt thành `package.json`, `pnpm-workspace.yaml` và `turbo.json`; thêm các script database/API/quality gate ở trên trước khi đánh dấu `F0-01`. Root `prettier.config.mjs` bật `prettier-plugin-tailwindcss` và cấu hình `tailwindStylesheet` theo Web/Admin, vì mỗi app sở hữu `globals.css` riêng. `.husky/pre-commit` gọi `pnpm exec lint-staged`; `.husky/install.mjs` chỉ import/chạy Husky khi có dev dependencies và không ở CI/production, tránh lỗi install runtime image. Hook chỉ format file source đã stage, không thay CI: `pnpm format:check` nằm trong `pnpm check` và chạy trong pull request. [Prettier pre-commit](https://prettier.io/docs/precommit), [Tailwind plugin](https://github.com/tailwindlabs/prettier-plugin-tailwindcss#readme), [Husky](https://typicode.github.io/husky/get-started.html). Server TS dùng `module/moduleResolution: NodeNext`, `experimentalDecorators` và `emitDecoratorMetadata` cho Nest; Next dùng `moduleResolution: Bundler` theo generator. Mọi workspace bật `strict` và `noUncheckedIndexedAccess`. API/worker xuất ESM thật, không trộn CommonJS output. Cài dependency theo bảng, tạo lockfile, rồi dùng `pnpm install --frozen-lockfile` trong CI. Cài browser E2E một lần bằng `pnpm exec playwright install chromium` và engine thứ hai được chọn cho release.

```js
// prettier.config.mjs
export default {
  plugins: ['prettier-plugin-tailwindcss'],
  overrides: [
    { files: 'apps/web/src/**/*.{ts,tsx,js,jsx}', options: { tailwindStylesheet: './apps/web/src/app/globals.css' } },
    { files: 'apps/admin/src/**/*.{ts,tsx,js,jsx}', options: { tailwindStylesheet: './apps/admin/src/app/globals.css' } },
  ],
};
```

```js
// .husky/install.mjs
if (process.env.CI === 'true' || process.env.NODE_ENV === 'production') process.exit(0);
const husky = (await import('husky')).default;
husky();
```

```sh
# .husky/pre-commit
pnpm exec lint-staged
```


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
