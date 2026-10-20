---
doc_id: ADR-0013
title: Zustand UI state, shadcn/ui Radix và Tailwind cho Web/Admin
status: accepted
owner: Product & Engineering
last_updated: 2026-10-20
version: 1.0.1
---

# ADR-0013 — Frontend state, UI và format source

## Bối cảnh

Web và Admin cần UI component thống nhất, source có thể tùy biến, và một cách giữ trạng thái giao diện khi nhiều component trong cùng luồng phối hợp. Blueprint trước chọn React local/React Query/URL nhưng loại Zustand; tài liệu cũng chưa chốt shadcn/ui hoặc Git hook format source.

## Quyết định

- Web và Admin dùng `zustand` cho **client UI state dùng chung trong feature/widget**. Store đặt theo owner trong `model`, khởi tạo ở Client Component boundary/provider khi state phụ thuộc request/session/route. Không đọc/ghi store từ Server Component, không dùng singleton server store xuyên request. React local giữ state một component, React Hook Form giữ form, URL giữ locale/workspace/shop/filter/cursor có thể chia sẻ; React Query giữ API query/mutation/cache. PostgreSQL và API vẫn là nguồn nghiệp vụ/quyền. Không lưu session, capability, cart/order/payment/stock/giá/quote hay bản sao response trong Zustand. Reset UI store, form và private query cache khi đổi actor/workspace/shop hoặc logout. [Zustand Next.js guide](https://zustand.docs.pmnd.rs/learn/guides/nextjs).
- Web/Admin dùng shadcn/ui bản **Radix UI**; vị trí source và quy tắc tạo component riêng đã được [ADR-0014](ADR-0014-app-owned-shadcn-components.md) thay thế. Pin CLI, chỉ thêm component cần dùng và review source sinh ra như code thường.
- Tailwind CSS và semantic token là cách style của Web/Admin. Feature/page/component dùng utility classes; không viết CSS thuần, CSS Modules hoặc một hệ thống style song song. CSS global chỉ giữ Tailwind import, token CSS variables/theme và foundation thật sự cần thiết. Giữ keyboard/focus/contrast/responsive contract trong [design system](../system/design-system.md).
- Prettier là formatter duy nhất, có `prettier-plugin-tailwindcss` với stylesheet Tailwind 4. Root Husky `pre-commit` chạy lint-staged để format staged source; root `format`/`format:check` áp dụng toàn bộ `apps/**/src` và `packages/**/src`. CI chạy `format:check` độc lập vì local hook có thể không chạy. ESLint/typecheck/test vẫn là gate riêng. [Prettier pre-commit](https://prettier.io/docs/precommit), [Husky](https://typicode.github.io/husky/get-started.html).

## Hệ quả triển khai

F0 scaffold cần hai `components.json` trong Web/Admin, Tailwind theme entry ở mỗi app, root Prettier config, Husky hook và lint-staged. Web/Admin thêm store theo nhu cầu của feature, có reset/hydration test; không cần tạo store rỗng cho mọi route. Review generated component và kiểm accessibility/visual state ở hai app; kiểm `format:check` trong CI. Các chi tiết setup thuộc [stack](../platform/stack-dependencies.md), [local development](../platform/local-development.md), [Web](../apps/web/README.md) và [Admin](../apps/admin/README.md).
