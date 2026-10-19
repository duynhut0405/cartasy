---
doc_id: APP-WEB
title: Web Frontend — multilingual storefront
status: accepted
owner: Storefront Engineering
last_updated: 2026-10-19
version: 4.4.0
source_of_truth: Setup, src, locale routing, design composition, state và integration của apps/web
---

# Web Frontend

[Preview thư viện, module và state/store của toàn workspace](../source-preview.md).

## Vai trò và stack

`apps/web` là Next.js 16 App Router/React 19/TypeScript 6 storefront, chạy Node server. F1 có giao diện tiếng Việt và tiếng Anh bằng `next-intl`; URL luôn có `/vi` hoặc `/en`, mặc định `vi`. Locale đổi ngôn ngữ giao diện/nội dung, không đổi thị trường Việt Nam, giá VND hoặc bản chất đơn hàng mô phỏng. [ADR-0012](../../decisions/ADR-0012-multilingual-storefront.md). Thư viện trực tiếp: `next`, `react`, `react-dom`, `next-intl`, `zustand`, `@tanstack/react-query`, `openapi-fetch` + generated type từ `@cartasy/contracts`, `react-hook-form`/`zod`/`@hookform/resolvers`, `tailwindcss`, `@cartasy/ui` (component shadcn/ui bản Radix) và `lucide-react`. Dev: `eslint-config-next`, `@types/react`, `@playwright/test` tại test workspace; Prettier/Husky/lint-staged ở root. Version/policy ở [stack](../../platform/stack-dependencies.md), [ADR-0013](../../decisions/ADR-0013-frontend-state-ui-tooling.md).

Web chỉ hiển thị và lấy dữ liệu qua API; không import `@cartasy/server` hoặc `@cartasy/database`. Quy tắc nghiệp vụ ở [domains](../../domains/README.md), route/payload ở [API contracts](../../system/api-contracts.md), UI tokens ở [design system](../../system/design-system.md).

## `src` chuẩn

```text
next.config.ts             # createNextIntlPlugin
components.json            # shadcn CLI aliases; ui trỏ @cartasy/ui
messages/
  vi/*.json                 # Common, Navigation, Catalog, Cart, Checkout, Orders,
                            # Account, Notifications, Simulation, Metadata, Errors...
  en/*.json                 # cùng namespace/key/ICU placeholder với vi
public/
  sw.js                     # Web Push service worker
src/
  proxy.ts                  # next-intl locale negotiation; exclude API/static/SW
  i18n/
    routing.ts              # locales, defaultLocale, prefix, pathnames tĩnh
    request.ts              # validated locale, load namespace messages
    navigation.ts           # localized Link/redirect/router
    formats.ts              # locale-aware date/number; tiền vẫn VND
  app/
    globals.css            # chỉ Tailwind import/theme token entry, không viết CSS cho feature
    [locale]/
      layout.tsx            # root html lang, provider, font, import global Tailwind entry
      loading.tsx
      error.tsx
      not-found.tsx
      (store)/
        layout.tsx          # shell: header, navigation, footer
        page.tsx
        categories/[slug]/page.tsx
        products/[slug]/page.tsx
        shops/[slug]/page.tsx
        search/page.tsx
        pages/[slug]/page.tsx
      (account)/
        layout.tsx          # session UX; API vẫn xác thực từng request
        cart/page.tsx
        checkout/page.tsx
        orders/page.tsx
        orders/[id]/page.tsx
        wishlist/page.tsx
        reviews/page.tsx
        returns/page.tsx
        support/page.tsx
        support/[id]/page.tsx
        notifications/page.tsx
        account/page.tsx
      (auth)/sign-in/page.tsx
      (auth)/sign-up/page.tsx
      (auth)/verify-email/page.tsx
      (auth)/recovery/page.tsx
    sitemap.ts              # published locale URLs + hreflang
    robots.ts
  features/
    locale-switcher/
    discovery/               # search/filter/url sync
    product-detail/          # variant/listing selector
    cart/                    # guest/server cart and merge
    checkout/                # quote, confirmation, idempotency
    payment-result/          # pending/success/failure UI
    order-history/
    wishlist/
    reviews/
    returns/
    support/
    account/
    notifications/
    push-permission/
  entities/
    product/
    listing/
    shop/
    cart/
    order/
    shipment/
  widgets/
    site-header/
    site-footer/
    category-navigation/
    product-grid/
    product-detail-view/
    order-summary/
  shared/
    api/                     # typed client, error mapper, query keys
    providers/               # React Query provider; scoped Zustand provider khi cần
    realtime/                # SSE + reconnect/poll fallback
    auth/                    # session hook, route gate
    config/                  # public env parser
    ui/                      # app composition of @cartasy/ui
    lib/                     # domain-free date/money formatting
tests/                      # app unit/component; E2E ở workspace tests/e2e
```

Đây là blueprint, không yêu cầu tạo thư mục rỗng trước khi làm feature. `app` chỉ route/layout/metadata và compose widget/feature; `features` sở hữu thao tác người dùng; `entities` giữ presentation model/component nhỏ; `widgets` là tổ hợp section; `shared` không chứa rule domain. Dependency `app → widgets/features → entities → shared`; feature khác nhau giao tiếp qua composition ở route/widget hoặc public API của owner. `packages/ui` sở hữu token và component shadcn/ui bản Radix; `shared/ui` chỉ adapter cho Web. Store Zustand nằm trong `features/<owner>/model` hoặc `widgets/<owner>/model`, chỉ tạo provider ở biên Client Component phù hợp; không tạo store tổng hợp chứa mọi feature. Không tạo `services`/`utils` chung hay copy business rule.

### Mẫu vertical slice: product detail

```text
app/[locale]/(store)/products/[slug]/page.tsx        # fetch theo locale, metadata, compose
widgets/product-detail-view/ui/ProductDetailView.tsx  # bố cục ảnh, thông tin, listing
features/product-detail/ui/ListingSelector.tsx        # chọn variant/listing, add-to-cart
features/product-detail/model/selection.ts            # UI state, không tính giá
entities/product/ui/ProductGallery.tsx                # approved media và alt
entities/listing/ui/ListingPrice.tsx                  # giá API gắn đúng shop
shared/api/catalog.ts                                 # typed transport adapter
```

Mỗi slice tạo `ui/`, `model/`, `api/` khi cần và `index.ts` public export; không áp một bộ thư mục rỗng cho mọi feature. Route không chứa form logic hoặc SQL. `api/` của feature chỉ map contract đã định nghĩa, không nhân bản HTTP client. Rule giá, stock, publish và quyền nằm ở domain/API.

## Route đa ngôn ngữ, nội dung và SEO

`routing.ts` khai báo `locales: ['vi', 'en']`, `defaultLocale: 'vi'`, `localePrefix: 'always'`. Next.js 16 dùng `src/proxy.ts` với `createMiddleware(routing)` để xử lý `/` và locale negotiation; matcher bỏ qua `/api`, `_next`, file tĩnh và service worker. `src/i18n/request.ts` lấy locale từ `next/root-params`, validate bằng allowlist rồi tải namespace message; `next.config.ts` đăng ký `createNextIntlPlugin`. `[locale]/layout.tsx` validate locale, khai báo `generateStaticParams` cho route cần static render, `<html lang>` và `NextIntlClientProvider`; Server Components/metadata dùng `getTranslations`. `setRequestLocale` là API legacy nên không dùng trong baseline Next 16.4. Link/navigation dùng helper `next-intl`, không nối URL thủ công. [Routing setup](https://next-intl.dev/docs/routing/setup), [configuration](https://next-intl.dev/docs/routing/configuration), [request setup](https://next-intl.dev/docs/usage/configuration).

UI copy là ICU message có namespace và placeholder/plural, không nối câu bằng string; nhãn “mô phỏng” có bản dịch tương đương. CI kiểm key/placeholder parity giữa `vi` và `en`. Nội dung catalog/shop/CMS/alt text nằm ở DB của owner, **không** ở `messages`. Locale switcher giữ cùng trang khi bản dịch và slug đích đã publish; nếu thiếu thì về category/home cùng locale với thông báo rõ. Chỉ giữ query allowlist; cart/session/checkout attempt không tạo mới khi đổi ngôn ngữ.

Public API nhận locale URL một cách tường minh, cache key/tag chứa locale. Product/category/shop/CMS chưa có bản dịch published trả `404` cho URL locale đó, không index trang tiếng Anh với nội dung tiếng Việt giả. Canonical và hreflang chỉ nối bản dịch đã publish; sitemap có alternate URLs, search theo policy canonical/noindex, private routes luôn `noindex`. Metadata/OG/structured data dùng cùng locale với nội dung và không tạo `Offer`/rating schema gây hiểu nhầm giao dịch thật. [Catalog](../../domains/catalog.md), [CMS](../../domains/cms-content.md), [API](../../system/api-contracts.md), [Next metadata](https://nextjs.org/docs/app/api-reference/functions/generate-metadata), [sitemap](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap).

## Tổ chức thiết kế trang

Thiết kế theo bốn tầng: `packages/ui` cho foundation/token/component shadcn/ui bản Radix; `entities` cho product/listing/shop visuals; `features` cho control và feedback; `widgets` cho section hoàn chỉnh; route chọn layout, thứ tự section và metadata. Trang product ghép `ProductGallery`, `ListingSelector`, `ShopIdentity`, `SimulationNotice`; giá luôn gắn listing/shop. Header/footer, breakpoint và token không sao chép vào từng page. UI control/primitive bắt đầu từ component shadcn/ui trong `@cartasy/ui`; domain component tự compose từ các primitive đó. Dùng Tailwind utilities và semantic token cho layout/visual, không viết CSS thuần hoặc CSS Modules trong feature/page; file CSS global chỉ dành cho Tailwind import và token/theme nền tảng. [Design system](../../system/design-system.md) sở hữu responsive grid, typography, spacing, component states, nội dung và accessibility.

Mỗi route có loading/empty/error/success, unauthenticated/forbidden, stale price/stock và thiếu bản dịch; review mobile 320–390 px, tablet, desktop, keyboard/focus/screen reader/reduced motion. Kiểm cả `vi` và `en` với chuỗi dài, wrap nút/menu/filter/dialog và giá VND. Ảnh dùng approved media, aspect ratio theo placement và alt text đúng locale; ảnh lỗi có fallback. Sponsored/affiliate sau MVP có disclosure đúng locale.

## Routing, SSR và state

Order/cart snapshot và user-generated content có thể giữ ngôn ngữ gốc của thời điểm tạo; Web hiển thị nhãn “nội dung gốc” khi khác locale UI, không dịch máy hoặc sửa snapshot. API `locale` chỉ chọn bản dịch public read; đổi locale không ảnh hưởng ID giỏ, order, giá hoặc idempotency key.

Public home/category/product/shop/search/CMS page dùng Server Components cho SEO/metadata và fetch public catalog theo locale với cache/tag + revalidation từ API; dữ liệu giá/tồn kho có TTL ngắn và thông báo có thể thay đổi. Search query/filter/sort/cursor nằm trong URL, canonical URL/noindex theo [discovery](../../domains/discovery.md). Private cart/account/order/wishlist/review/return/support dùng Client Components + typed API `credentials: include`; API cookie host-only không có ở Next origin, vì vậy không SSR dữ liệu riêng bằng cookie giả định. Session gate chuyển login rồi quay về URL hợp lệ; server API vẫn quyết định quyền.

React Query quản lý query/mutation, key gồm actor/shop/resource/filter/locale khi response có text; invalidate sau mutation; cache private được clear khi logout/account switch. Zustand giữ **client UI state dùng chung giữa nhiều component** trong cùng feature/widget, ví dụ mobile navigation hoặc gallery selection; state chỉ thuộc một component (dialog/tab đơn giản) ở React local, form ở React Hook Form, search/filter/sort/cursor ở URL. Store tạo trong Client Component boundary/provider khi có state theo session/route; Server Component không đọc/ghi Zustand và không tạo singleton server store dùng chung request. Guest cart dùng `localStorage` adapter versioned trong feature cart, chỉ chứa listing ID/quantity, merge idempotent khi login; không lưu giá/PII. Không sao chép cart/order/price/quote/session/consent từ API vào Zustand. Checkout lưu opaque idempotency key của mỗi attempt trong `sessionStorage` theo actor/cart attempt qua refresh/timeout, lookup theo key rồi mới clear sau kết quả terminal; không lưu quote/payment detail tại đây. Không tính giá cuối cùng ở browser. [Zustand với Next.js](https://zustand.docs.pmnd.rs/learn/guides/nextjs).

## API, lỗi và UI

Tạo `shared/api/client.ts` từ OpenAPI generated type, base URL từ env, include credential, request ID; parse error envelope theo `code`, abort stale queries. 401 → login/reauth, 403 → trạng thái không đủ quyền, 404 → not-found, 409 price/cart/stock → refetch quote và yêu cầu xác nhận lại, 429 → retry sau hạn, 5xx → retry có giới hạn và hiển thị request ID. Không retry checkout với key mới. Mọi luồng có loading/empty/error/success; focus management, keyboard, screen reader, mobile layout, alt text và nhãn “mô phỏng” ở payment/shipping/refund/review. Analytics theo [analytics](../../system/analytics.md), không gửi PII. Inbox/order badge dùng SSE hint (`EventSource` credentialed) và refetch typed API; reconnect bằng cursor, fallback polling. Web Push dùng service worker + Push API/VAPID public key, xin permission sau hành động người dùng, đăng ký/hủy qua API. Không chứa VAPID private key hoặc dữ liệu order nhạy cảm trong push payload; trên iOS hướng dẫn thêm web app vào Home Screen. [Communications](../../domains/communications.md) sở hữu consent/campaign.

## Env và setup độc lập

`NEXT_PUBLIC_API_URL=http://localhost:3002/v1`, `NEXT_PUBLIC_SITE_URL=http://localhost:3000`, `NEXT_PUBLIC_VAPID_PUBLIC_KEY` là public; không đặt secret trong `NEXT_PUBLIC_*`. `PORT=3000` local; URL production được inject tại build và phải khớp CORS/API cookie site. Dùng chung Node/pnpm/lockfile và dependency services tại [local development](../../platform/local-development.md).

1. Từ workspace root, pin runtime, `pnpm install --frozen-lockfile` và bật PostgreSQL/Mailpit/object store; chạy migration/seed/API theo [local setup](../../platform/local-development.md).
2. Tạo `apps/web/package.json` với dependencies trên và script `dev`/`build`/`start`/`lint`/`typecheck`/`test`. Cấu hình `next-intl` plugin/proxy/routing/messages, `components.json` alias tới `@cartasy/ui`, Tailwind source/theme entry, tạo `src` theo từng vertical slice, TS strict, import alias chỉ trong app. Root Prettier/Husky format source theo [local setup](../../platform/local-development.md).
3. Tạo `.env.local` từ tên biến trên; chạy `pnpm --filter @cartasy/web dev` (script bind port 3000), xác nhận API CORS/cookie cho web origin.
4. Kiểm `/` redirect, `/vi`/`/en`, locale switch/deep link, SEO/404, rồi mở home, product, cart, sign-in, checkout; kiểm tra email verification trong Mailpit và order mock success/failure. Lệnh gate: `pnpm --filter @cartasy/web lint`, `typecheck`, `build`; E2E storefront ở test workspace.

Luồng phát triển: cập nhật domain/API contract trước → generated client → feature/entity/widget → route → accessibility/E2E → analytics; review bản desktop và mobile. Không ghi “đã test” nếu chưa chạy lệnh/bằng chứng. MVP phải kiểm mixed-shop cart/checkout và notification đúng buyer; không coi hai-shop là F2.

Sau MVP, sponsored card/banner và external affiliate offer là feature UI riêng có nhãn theo [Design](../../system/design-system.md). Affiliate CTA thông báo rời Cartasy để mua thật trên đối tác, không gọi `POST /v1/checkouts` và không thêm `externalOfferId` vào cart. Không dựng route/nút F3 trong F1 khi partner/rights/consent chưa được duyệt. [Affiliate](../../domains/affiliate.md), [Advertising](../../domains/advertising.md).
