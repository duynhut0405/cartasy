---
doc_id: EXEC-FRONTEND-FIRST-SETUP
title: Cartasy Database-first, Web-priority Setup Progress Checklist
status: active
owner: Product & Engineering
last_updated: 2026-10-20
version: 1.1.0
source_of_truth: Tiến độ setup workspace theo chiến lược PostgreSQL/API thật trước, ưu tiên hoàn thiện Web theo vertical slice
review_cycle: Sau mỗi phase setup hoặc khi đổi chiến lược API, schema hay thứ tự vertical slice
---

# Checklist setup Database-first, ưu tiên Web

Checklist này đưa Cartasy từ repository chỉ có tài liệu tới vertical slice Web đầu tiên chạy trên **PostgreSQL và API thật ngay từ đầu**. Thứ tự chuẩn của mỗi slice là:

```text
Domain invariant
  → schema + SQL migration + seed local/test
  → repository/application use case
  → controller/DTO/OpenAPI
  → generated typed client
  → Web UI
  → integration/E2E evidence
```

Không xây mock backend hoặc runtime fixture adapter để rồi thay thế. Dữ liệu seed trong PostgreSQL và test fixture cho database cô lập vẫn bắt buộc; đây là dữ liệu phát triển/kiểm thử, không phải nguồn dữ liệu giả nằm trong Web hay một API tạm.

Nguồn chuẩn vẫn là [Documentation Index](../README.md), [Source Architecture](../system/source-architecture.md), [Data Model](../system/data-model.md), [API Contracts](../system/api-contracts.md), [Local Development](../platform/local-development.md), [Testing](../platform/testing-quality.md) và [Build Checklist](build-checklist.md). File này chỉ chia nhỏ thứ tự thực thi; một mục được đánh dấu ở đây không tự động hoàn thành mục F0/F1 tương ứng trong Build Checklist.

## 1. Quy tắc cập nhật tiến độ

- Chỉ chuyển `[ ]` thành `[x]` khi có commit hoặc artifact và lệnh kiểm tra đã chạy thành công.
- Mỗi phase chỉ là `DONE` khi toàn bộ mục bắt buộc và gate của phase đã đạt.
- Nếu bị chặn, ghi `BLOCKED: lý do; owner; quyết định cần có` ngay dưới mục liên quan.
- Web chỉ nhận dữ liệu nghiệp vụ qua typed API client; không import seed, database row, server entity hoặc hardcode business response trong component.
- API GET phục vụ Web phải đọc PostgreSQL qua repository/read model thật; không có `/mock/*`, `/fake/*` hoặc nhánh controller trả JSON tạm.
- Test double chỉ được dùng ở tầng unit phù hợp. Integration test và E2E của slice phải dùng PostgreSQL thật, migration thật và API thật.
- Chỉ đánh dấu slice hoàn thành khi đạt trạng thái `INTEGRATED`: schema, migration, API, Web và test đều có bằng chứng.

## 2. Bảng trạng thái phase

Cập nhật cột trạng thái bằng một trong `NOT_STARTED`, `IN_PROGRESS`, `BLOCKED`, `DONE`.

| Phase | Kết quả | Trạng thái | Bằng chứng gần nhất |
|---|---|---|---|
| S0 | Phạm vi và thứ tự delivery được ghi nhận | NOT_STARTED | — |
| S1 | Root workspace và quality tooling chạy được | NOT_STARTED | — |
| S2 | App/package skeleton đúng dependency boundary | NOT_STARTED | — |
| S3 | PostgreSQL và migration/seed foundation chạy được | NOT_STARTED | — |
| S4 | Schema Public Catalog được migrate và seed | NOT_STARTED | — |
| S5 | Public Catalog GET API và typed client chạy trên DB thật | NOT_STARTED | — |
| S6 | Web foundation `/vi`/`/en` và design system hoạt động | NOT_STARTED | — |
| S7 | Public Catalog vertical slice đạt `INTEGRATED` | NOT_STARTED | — |
| S8 | Pipeline sẵn sàng mở rộng sang auth/cart/checkout | NOT_STARTED | — |

## S0 — Chốt phạm vi setup

- [ ] `S0-01` Xác nhận invariant: multi-seller F1, Buyer commerce mô phỏng, VND integer/string API, một Admin app, Web `/vi`/`/en`.
- [ ] `S0-02` Xác nhận chiến lược delivery: PostgreSQL/API thật trước từng màn hình, Web là ưu tiên sản phẩm nhưng không đi trước contract và persistence của slice.
- [ ] `S0-03` Chọn vertical slice đầu tiên là public discovery: category/search → product → listing theo shop → shop page; home/CMS có thể nối sau khi Catalog read path ổn định.
- [ ] `S0-04` Ghi rõ phần ngoài phạm vi slice đầu: auth thật, cart server, checkout transaction, payment settlement và seller/admin workflow.
- [ ] `S0-05` Chốt không xây mock runtime data source; local/test dùng PostgreSQL seed đã gắn nhãn và database cô lập.
- [ ] `S0-06` Nếu thêm dependency ngoài stack hiện hành, cập nhật tài liệu owner và pin exact version trước khi cài.

Gate S0:

- [ ] Không có quyết định mở nào chặn public Catalog slice.
- [ ] Owner, entity, endpoint, response/error và tiêu chí hoàn thành của slice đã rõ.

## S1 — Root workspace và tooling

- [ ] `S1-01` Tạo root `package.json` private ESM với `packageManager` và Node engine đúng baseline đã xác minh.
- [ ] `S1-02` Tạo `.node-version`, `pnpm-workspace.yaml` và `turbo.json`.
- [ ] `S1-03` Tạo TypeScript base config với `strict` và `noUncheckedIndexedAccess`.
- [ ] `S1-04` Cấu hình ESLint, dependency-cruiser, no-cycle và cấm deep import `@cartasy/*/src/*`.
- [ ] `S1-05` Cấu hình Prettier, Tailwind plugin, `format` và `format:check`.
- [ ] `S1-06` Cấu hình Husky/lint-staged; hook local không thay thế CI.
- [ ] `S1-07` Tạo root scripts `dev`, `build`, `lint`, `typecheck`, `test`, `test:integration`, `test:e2e` và `check`.
- [ ] `S1-08` Cài dependency, sinh và commit một `pnpm-lock.yaml`; CI dùng frozen lockfile.
- [ ] `S1-09` Thêm `.gitignore` và `.env.example` không chứa secret thật.

Gate S1:

- [ ] `pnpm install --frozen-lockfile` thành công trên workspace sạch.
- [ ] `pnpm format:check`, `pnpm lint` và `pnpm typecheck` chạy được, không dùng `--passWithNoTests` hoặc tắt rule để tạo kết quả xanh.

## S2 — App và package skeleton

- [ ] `S2-01` Tạo bốn app: `apps/web`, `apps/admin`, `apps/api`, `apps/worker`.
- [ ] `S2-02` Tạo các package: `server`, `database`, `contracts`, `config`, `ui`, `observability`, `testkit`.
- [ ] `S2-03` Mỗi workspace có `package.json`, public `exports` và scripts tối thiểu phù hợp.
- [ ] `S2-04` `apps/web` và `apps/admin` không import `server` hoặc `database`.
- [ ] `S2-05` `packages/server` không import Next, Nest hoặc Drizzle.
- [ ] `S2-06` `packages/database` chỉ triển khai public port của `server`; không export DB client cho frontend/controller.
- [ ] `S2-07` `packages/testkit` chỉ là dev dependency và không đi vào production runtime/bundle.
- [ ] `S2-08` Tạo README ngắn cho từng workspace theo source thực tế, không copy blueprint như bằng chứng triển khai.

Gate S2:

- [ ] Cả bốn app build độc lập dù Admin/Worker mới chỉ là skeleton.
- [ ] Architecture test chặn import ngược, cycle và deep import bằng một test case vi phạm có kiểm chứng.

## S3 — PostgreSQL, migration và environment foundation

- [ ] `S3-01` Tạo Compose cho PostgreSQL, Mailpit và S3-compatible object store với image pin và healthcheck.
- [ ] `S3-02` Chốt cổng local: Web `3000`, Admin `3001`, API `3002`, PostgreSQL `5432`, Mailpit `1025/8025`, object store `9000/9001`.
- [ ] `S3-03` Dùng thống nhất `localhost`; không trộn với `127.0.0.1` trong cookie/CORS/auth URL.
- [ ] `S3-04` Tạo typed env parser theo process; thiếu biến bắt buộc phải fail-fast.
- [ ] `S3-05` Tách public env khỏi server secret; không có database, SMTP, S3 secret trong `NEXT_PUBLIC_*`.
- [ ] `S3-06` Tạo Drizzle client/pool và UnitOfWork/transaction foundation trong `packages/database`.
- [ ] `S3-07` Tạo migration runner có history/checksum/advisory lock; không dùng `drizzle-kit push` cho shared environment.
- [ ] `S3-08` Tạo scripts `db:generate`, `db:migrate`, `db:seed`, `db:check`.
- [ ] `S3-09` Tạo API `/health/live` và `/health/ready`; readiness kiểm PostgreSQL và dependency bắt buộc.
- [ ] `S3-10` Tạo database test cô lập; test không dùng database local đang phục vụ developer.

Gate S3:

- [ ] Compose dependency healthy và quy trình start/reset được ghi rõ.
- [ ] Một migration rỗng/baseline apply được trên DB sạch và được migration history ghi nhận đúng một lần.
- [ ] API readiness đổi đúng khi PostgreSQL sẵn sàng hoặc mất kết nối.
- [ ] Secret scan không phát hiện credential thật trong repository.

## S4 — Schema, migration và seed Public Catalog

- [ ] `S4-01` Xác định owner/schema tối thiểu của slice: `shops`, `shop_translations`, `categories`, `category_translations`, `brands`, `products`, `product_translations`, `product_variants`, `listings`, `inventory_stocks`, media và localized alt text cần thiết.
- [ ] `S4-02` Không tạo bảng F2/F3, ad billing, affiliate hoặc module placeholder chưa triển khai.
- [ ] `S4-03` Tiền lưu `bigint amount_minor` + `currency`; F1 chỉ `VND`.
- [ ] `S4-04` Thêm FK, CHECK, unique/version và index theo invariant/query thật; listing luôn có `shop_id`.
- [ ] `S4-05` Localized slug unique theo locale; resource và translation đều phải published mới được public.
- [ ] `S4-06` Viết SQL migration đã review; không sửa migration đã apply.
- [ ] `S4-07` Seed local/test idempotent có hai seller, hai shop active, category/product/variant và hai listing cùng variant với giá/stock khác nhau.
- [ ] `S4-08` Seed có bản `vi`, ít nhất một bản `en` published và một bản `en` draft/unpublished.
- [ ] `S4-09` Seed có listing active, paused và hết hàng để Web kiểm trạng thái thật từ DB.
- [ ] `S4-10` Seed không tạo fake review, fake sold count hoặc order giả như sự kiện thật.
- [ ] `S4-11` Tạo repository/read model Public Catalog, map database row sang application result; controller không nhận Drizzle row.
- [ ] `S4-12` Integration test constraint, localized slug, publication eligibility, shop isolation và query plan trên PostgreSQL thật.

Gate S4:

- [ ] `db:migrate` + `db:seed` chạy lặp an toàn trên local/test theo policy.
- [ ] Migration chạy được từ DB sạch và upgrade fixture phiên bản trước.
- [ ] Seed chứng minh được hai shop độc lập và nội dung `vi`/`en` theo đúng publish state.

## S5 — Public Catalog API và typed client

- [ ] `S5-01` Tạo domain/application query/port cho category, search, product, listing và shop public read.
- [ ] `S5-02` Tạo Nest controller/DTO/mapper production; controller không chứa SQL hoặc eligibility rule.
- [ ] `S5-03` Public GET yêu cầu `locale=vi|en`, trả `Content-Language` và chỉ trả resource/translation published.
- [ ] `S5-04` Money response dùng string minor unit và `currency: "VND"`; không dùng JSON float.
- [ ] `S5-05` List dùng cursor/limit, filter/sort allowlist và deterministic tie-breaker.
- [ ] `S5-06` Implement endpoint tối thiểu: `/v1/categories`, `/v1/products`, `/v1/products/:slug`, `/v1/products/:slug/listings`, `/v1/shops/:slug`, `/v1/shops/:slug/products`.
- [ ] `S5-07` Error envelope chuẩn cho invalid locale/cursor, unpublished detail, listing unavailable và internal error.
- [ ] `S5-08` Sinh OpenAPI artifact từ controller/DTO và generated TypeScript client; cấm sửa generated file bằng tay.
- [ ] `S5-09` Tạo API contract test cho locale, money, cursor, published filter và stable error code.
- [ ] `S5-10` Tạo HTTP/PostgreSQL integration test bằng seed database thật.
- [ ] `S5-11` API không có route hoặc config runtime trả mock catalog data.

Gate S5:

- [ ] `pnpm api:generate` và `pnpm api:check` chạy xanh.
- [ ] Curl/API test đọc được dữ liệu hai shop từ PostgreSQL qua repository/application/controller thật.
- [ ] Unsupported locale, unpublished translation, invalid cursor và cross-scope resource có kết quả đúng contract.

## S6 — Web foundation

- [ ] `S6-01` Cấu hình Next App Router và `next-intl` với locale prefix luôn có `/vi`/`/en`, mặc định `vi`.
- [ ] `S6-02` Cấu hình `src/proxy.ts`, routing/request/navigation helpers và locale validation.
- [ ] `S6-03` Tạo message namespace `vi`/`en` và gate kiểm key/ICU placeholder parity.
- [ ] `S6-04` Tạo `@cartasy/ui` bằng shadcn/ui bản Radix, Tailwind semantic token và public export có chủ đích.
- [ ] `S6-05` Tạo Web shell, skip link, header/navigation/footer, responsive container và focus styles.
- [ ] `S6-06` Cấu hình React Query cho private/server state tương lai và error mapping từ API envelope.
- [ ] `S6-07` Zustand chỉ giữ UI state dùng chung theo feature/widget; không lưu catalog response, giá, stock, session, cart hoặc quyền.
- [ ] `S6-08` URL là nguồn state cho locale/search/filter/sort/cursor.
- [ ] `S6-09` Typed API client dùng generated contract, base URL từ public env và hỗ trợ abort stale request.
- [ ] `S6-10` Public Server Component fetch API thật với cache key/tag có locale; dữ liệu giá/stock dùng TTL phù hợp và checkout sau này vẫn đọc primary.
- [ ] `S6-11` Có component/pattern cho loading, empty, recoverable error, forbidden, not found và unavailable translation.

Gate S6:

- [ ] `/` chuyển locale đúng; `/vi` và `/en` render độc lập.
- [ ] Web build không import server, database hoặc testkit.
- [ ] Keyboard, focus, contrast và layout 320–390 px được kiểm tra cho shell/component nền.

## S7 — Public Catalog vertical slice Integrated

- [ ] `S7-01` Category/search dùng API thật; query, filter, sort và cursor nằm trong URL.
- [ ] `S7-02` Product Detail hiển thị product → variant → listing → shop rõ ràng từ DB thật.
- [ ] `S7-03` Giá luôn gắn listing/shop và có nhãn “từ” khi là min price.
- [ ] `S7-04` Shop page chỉ hiển thị shop/listing public.
- [ ] `S7-05` Mỗi route có loading, empty, error, retry, not-found và unavailable state.
- [ ] `S7-06` Missing/unpublished translation trả 404; không fallback âm thầm từ `en` sang `vi`.
- [ ] `S7-07` Locale switch giữ trang khi localized slug published; fallback an toàn khi không có bản dịch.
- [ ] `S7-08` Metadata, canonical, hreflang và sitemap chỉ chứa locale đã publish.
- [ ] `S7-09` Responsive/mobile, keyboard, screen reader, ảnh fallback và alt text theo locale được kiểm tra.
- [ ] `S7-10` Không hiển thị fake sold count, fake review, fake stock certainty hoặc `Offer` schema gây hiểu nhầm giao dịch thật.
- [ ] `S7-11` E2E chạy Web → API → PostgreSQL cho hai locale và hai shop.
- [ ] `S7-12` OpenAPI/generated client không drift; không có type/response duplicate trong Web.

Gate S7 — `INTEGRATED`:

- [ ] Migration clean + upgrade, seed, integration test, lint, typecheck, API/Web build và E2E đều xanh.
- [ ] Hai-shop isolation, publication eligibility và unpublished translation không bị lộ ở API/search/Web.
- [ ] Artifact/screenshot/report và commit SHA được ghi trong nhật ký bằng chứng.
- [ ] Chỉ sau gate này mới cập nhật mục Catalog tương ứng trong Build Checklist.

## S8 — Sẵn sàng mở rộng sang auth, cart và checkout

- [ ] `S8-01` Chu kỳ cho slice tiếp theo vẫn là domain → schema/migration/seed → repository/use case → API/OpenAPI → Web → integration/E2E.
- [ ] `S8-02` Better Auth dùng PostgreSQL session thật; kiểm cookie API host-only, CORS, Origin/CSRF và revoke trước private Web.
- [ ] `S8-03` Guest cart chỉ lưu `listingId + quantity + schemaVersion`; không lưu giá hoặc PII.
- [ ] `S8-04` Server cart/wishlist dùng owner policy, schema/migration thật và generated contract.
- [ ] `S8-05` Checkout UI không được coi hoàn thành trước quote API, transaction PostgreSQL và authoritative price/stock/promotion check.
- [ ] `S8-06` Trước checkout đã có UnitOfWork, reservation, idempotency record, outbox và Worker settlement.
- [ ] `S8-07` Checkout giữ opaque attempt key qua timeout và lookup by-key; client không tạo key mới khi chưa biết kết quả.
- [ ] `S8-08` Test plan gồm stale quote, price/stock conflict, last-stock race, duplicate key, rollback và mixed-shop split.
- [ ] `S8-09` Payment success/failure/timeout do mock payment **domain adapter nội bộ theo đặc tả sản phẩm**, không phải mock database/API cho Web; scenario control chỉ local/test và fail-closed ngoài môi trường đó.

Gate S8:

- [ ] Backend/data contract của mỗi màn hình được hoàn thành trước hoặc cùng lúc Web sử dụng nó.
- [ ] Không có quyết định client-side nào thay thế server về quyền, giá, stock, discount hoặc order/payment state.
- [ ] Không phát sinh bước “migrate từ mock backend sang real backend”; chỉ có migration schema tiến hóa bình thường theo release.

## 3. Quality gate tổng cho setup

- [ ] `Q-01` `pnpm install --frozen-lockfile`.
- [ ] `Q-02` `pnpm format:check`.
- [ ] `Q-03` `pnpm lint` và architecture boundary check.
- [ ] `Q-04` `pnpm typecheck`.
- [ ] `Q-05` `pnpm test`.
- [ ] `Q-06` PostgreSQL integration test.
- [ ] `Q-07` OpenAPI/generated client drift check.
- [ ] `Q-08` Migration clean + upgrade check.
- [ ] `Q-09` Build độc lập Web/Admin/API/Worker.
- [ ] `Q-10` E2E Web → API → PostgreSQL cho slice đã triển khai.
- [ ] `Q-11` Secret/dependency scan.
- [ ] `Q-12` Documentation link/front-matter check.

## 4. Nhật ký bằng chứng

Thêm một dòng sau mỗi gate. Không ghi `PASS` nếu không có command/artifact tương ứng.

| Ngày | Phase/Gate | Commit SHA | Environment | Command hoặc artifact | Kết quả | Owner |
|---|---|---|---|---|---|---|
| — | — | — | — | — | — | — |

## 5. Snapshot hiện trạng ban đầu

Ngày 20/10/2026, repository chỉ có `README.md`, `AGENTS.md` và bộ tài liệu `docs/`; chưa có manifest, lockfile, app, package, migration, OpenAPI, CI hoặc source. Vì vậy toàn bộ mục setup ở trên bắt đầu ở trạng thái chưa hoàn thành. Blueprint hoặc checkbox trong tài liệu không được coi là bằng chứng chạy.
