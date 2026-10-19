---
doc_id: ADR-0012
title: Locale-prefixed storefront và nội dung dịch theo domain owner
status: accepted
owner: Product & Engineering
last_updated: 2026-10-19
version: 1.0.0
---

# ADR-0012 — Storefront đa ngôn ngữ

## Bối cảnh

Storefront public cần mở rộng ngôn ngữ và SEO. Cây `apps/web/src/app/(store)` cũ không biểu diễn locale; catalog/CMS có text và slug đơn ngữ. Trộn bản dịch UI với nội dung kinh doanh sẽ làm khó publish, cache và kiểm soát URL.

## Quyết định

- F1 Web có `vi` và `en`, default `vi`; tất cả storefront và buyer route dùng `/[locale]`, `localePrefix: 'always'`. Dùng `next-intl` cho route negotiation, message UI, formatting, navigation. Next.js 16 dùng `src/proxy.ts`. Không đổi VND, shipping/payment/order mô phỏng theo locale. [next-intl routing](https://next-intl.dev/docs/routing/configuration), [proxy](https://next-intl.dev/docs/routing/middleware).
- UI message theo namespace trong Web; catalog, shop, CMS, media alt text theo bảng/aggregate của owner và workflow draft → review → published cho mỗi locale. Slug theo locale, identity của resource dùng ID ổn định.
- Public API nhận locale explicit ở read endpoint; response/cache/SEO cùng locale. Chỉ index page có bản dịch published; các locale còn thiếu không được tạo trang dịch giả. Notifications/email chọn template theo locale đã lưu của recipient, fallback `vi` có kiểm soát.
- Admin vẫn là **một** `apps/admin` với `/seller/*` và `/platform/*`. Admin quản lý bản dịch trong cùng workflow/capability; UI Admin có thể dịch dần theo user preference, không cần locale prefix SEO.

## Hệ quả triển khai

F0 phải có routing/messages/component pattern và validation parity; F1 cần migration cho localized content, API contract, preview/review/publish theo locale, slug redirect, cache invalidation, hreflang/sitemap và E2E hai locale. Nội dung `vi` hiện có được backfill thành translation `vi` qua migration đã kiểm chứng; không tự sinh bản `en` hoặc sửa order snapshot. Chỉ triển khai bản dịch khi có quyền nội dung và người duyệt.

## Ranh giới

F1 vẫn nhắm thị trường Việt Nam/VND. Thêm locale khác, tiền tệ khác hoặc thị trường khác cần quyết định Product, chính sách thuế/vận chuyển/pháp lý và contract mới; không suy ra từ việc có `next-intl`.
