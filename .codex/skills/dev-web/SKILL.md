---
name: dev-web
description: "Xây storefront Buyer trong apps/web: discovery, product, cart, checkout, locale, SEO và typed API. Không dùng cho apps/admin."
---

# Dev Web

- Đọc `docs/apps/web/README.md`, domain spec của slice, `docs/system/api-contracts.md` và `docs/system/design-system.md`. Theo `docs/delivery/frontend-first-setup-checklist.md`: PostgreSQL/API thật trước từng màn hình; ưu tiên public discovery S0–S7.
- Dùng route `/vi` và `/en`, nội dung đã publish, SEO theo locale và typed client sinh từ OpenAPI. Không đặt business rule hoặc mock runtime data trong Web.
- Dùng shadcn/ui Radix trực tiếp trong app; chỉ compose element riêng khi thiếu. React Query giữ server state, URL giữ filter/cursor, Zustand chỉ giữ UI state dùng chung.
- Cart chỉ gửi ID/số lượng; quote và checkout tin server, giữ idempotency key khi retry. Ghi rõ đơn/thanh toán/giao hàng mô phỏng trong UX; kiểm luồng Web → API → PostgreSQL.
