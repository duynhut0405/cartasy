---
doc_id: EXEC-BUILD-CHECKLIST
title: Cartasy Build Checklist
status: accepted
owner: Product & Engineering
last_updated: 2026-10-19
version: 4.4.0
source_of_truth: Công việc triển khai, gate và bằng chứng
---

# Checklist triển khai

Đây là **kế hoạch**, không là bằng chứng ứng dụng đã tồn tại. Mục chỉ chuyển `[x]` khi có commit/artifact/test/environment, owner và ngày xác nhận. Khi bị chặn: `BLOCKED: reason; owner; decision`. [Audit](audit-and-open-decisions.md) ghi quyết định mở.

## F0 — Foundation

- [ ] `F0-01` Pin Node/pnpm/TypeScript/framework/runtime exact, workspace/lockfile, peer compatibility và **bốn process Web/Admin/API/Worker** build độc lập; Seller/Platform cùng `apps/admin`; Prettier + Tailwind plugin/Husky/lint-staged format source, CI `format:check`.
- [ ] `F0-02` Shared packages public exports, domain/application/adapter boundary, dependency-cruiser và no-cycle/no-deep-import gate.
- [ ] `F0-03` PostgreSQL/Mailpit/S3 local, typed env/secret, health/readiness, migration clean+upgrade và seed **hai shop/hai seller/hai buyer** có quyền media.
- [ ] `F0-04` Better Auth verification/reset/session revoke, MFA Seller/Platform Admin, RBAC/shop capability/ABAC, `GET /v1/portal/context`, CSRF/CORS/Origin và object-level test cùng Admin origin.
- [ ] `F0-05` OpenAPI/generated client/event schema version, error envelope, request/correlation ID, ETag/idempotency contract.
- [ ] `F0-06` UnitOfWork, transactional outbox/inbox dedupe, background jobs/lease/checkpoint, fake clock và dead-letter replay audit.
- [ ] `F0-07` CI frozen install, lint/typecheck/build/unit/integration/E2E/contract/migration/security/docs link; staging deploy và rollback path.
- [ ] `F0-08` Máy sạch cài từ lockfile, migrate/seed, bật bốn process, smoke auth/tenant/health/OpenAPI và Seller/Platform workspace.
- [ ] `F0-09` Web `[locale]`/`next-intl`/message namespace, shared shadcn/ui **Radix** + Tailwind token/primitive/layout cho Web/Admin, Zustand UI store theo owner và React Query server state; kiểm parity `vi`/`en`, URL `/` chuyển locale đúng, store/cache reset khi đổi actor/shop.

## F1 — Buyer/Seller/Admin Marketplace MVP

- [ ] `F1-SELLER` Seller application/review/activation theo phase đã duyệt, membership/invite/MFA, khu `/seller/*` trong Admin workspace, shop switch, suspension, two-shop isolation.
- [ ] `F1-CATALOG` Product/variant/category/media/listing/shop, moderation/provenance, seller listing/price/stock, public SEO/search/filter/sort/cursor.
- [ ] `F1-BUY` Wishlist guest/member, cart merge, quote, voucher cơ bản, multi-shop checkout transaction/idempotency và price/stock conflict UX.
- [ ] `F1-PAY` Mock success/failure/timeout/replay, stock/coupon commit-release, order/shop-order/snapshot, seller scoped processing, shipping/refund mô phỏng.
- [ ] `F1-SERVICE` Review trải nghiệm mô phỏng, return/refund, support ticket/attachment/SLA, seller scoped reply/proposal, Admin queues, audit và notification đúng người/shop.
- [ ] `F1-TRUST` Report/takedown/appeal, evidence privacy, moderator conflict guard và checkout/search reaction khi ẩn listing/shop.
- [ ] `F1-NOTI` Inbox/email/Web Push opt-in/SSE fallback cho order/shop; subscription lifecycle, retry/dedupe/no PII payload.
- [ ] `F1-CAMPAIGN` Product new/promo/system announcement approval, preference/consent, quiet hours/frequency cap, fanout batch/checkpoint/pause/report.
- [ ] `F1-CONTENT` CMS home/banner/page draft/review/schedule/publish/rollback; reference eligibility/cache invalidation/SEO/XSS.
- [ ] `F1-I18N` Backfill nội dung `vi`, product/category/shop/media/CMS translation theo owner, localized slug/redirect, API `locale`/cache/search, preview/review/publish trong Admin, SEO và template notification `vi`/`en`.
- [ ] `F1-REPORT` Seller/shop/Admin aggregate với `asOf`/lag/tenant scope, reconciliation và nhãn giá trị mô phỏng.
- [ ] `F1-UX` Mobile/WCAG 2.2 AA, no real payment/shipping claims, permission/empty/error/loading/timeout state trên Buyer/Seller/Admin.

## F1 — Release gate

- [ ] `GATE-01` Staging E2E hai seller/hai shop/hai buyer: mixed-shop order → mỗi seller đúng phần → mock ship/return/refund → buyer notification; cross-tenant/cross-role deny, identity hai role đổi workspace/shop không lộ cache cũ.
- [ ] `GATE-02` Race/retry/reconciliation checkout-stock-voucher-payment-refund/order/campaign; no oversell/duplicate delivery.
- [ ] `GATE-03` Auth/MFA/CSRF/CORS/PII/attachment/Web Push/SSE/campaign security review và consent/privacy/Terms/seller Terms được duyệt.
- [ ] `GATE-04` CMS/content quyền sử dụng, moderation, no fake reviews/sales/price; Web Push iOS fallback và email template wording.
- [ ] `GATE-I18N` E2E `/vi`/`/en` browse→cart→checkout→order, locale switch/deep link, thiếu bản dịch 404, canonical/hreflang/sitemap, cache separation, UI message parity và layout dài/mobile/a11y; có report/artifact.
- [ ] `GATE-05` Migration clean+upgrade, OpenAPI drift, code/docs links, backup/restore drill, load/SLO, alert recipient, rollback/killswitch trên staging.
- [ ] `GATE-06` Product và Engineering owner ký release record cho commit/artifact; blocker ở [Audit](audit-and-open-decisions.md) đã đóng hoặc có waiver có owner.

## F2 — Growth

- [ ] `F2-FLASH` Flash sale quota+stock atomic, pricing conflict, moderation, load/race.
- [ ] `F2-REC` Recommendation rule-based/consent fallback, eligibility/fairness/latency.
- [ ] `F2-BULK` Seller import có provenance/quota/preview/SSRF guard; advanced report và abuse automation.

## F3 — Monetization có điều kiện

- [ ] `F3-ADS` Sponsored Product/Shop/Search/Display slot có nhãn, eligibility, creative moderation, consent, viewability/click anti-fraud, tenant isolation và frequency/budget cap; ad serving không làm sai organic sort hoặc mock checkout.
- [ ] `F3-AD-BILLING` Chỉ khi M-03 được duyệt: advertiser contract/pricing, invoice/receipt/credit note/refund/ledger, provider webhook security, Finance capability, reconciliation và release gate riêng; không dùng Buyer `orders/payments` mô phỏng.
- [ ] `F3-AFF` Chỉ khi M-04/M-05 được duyệt: external offer có quyền nội dung, CTA ra đối tác có disclosure, domain allowlist/deep-link fallback, click consent, partner report/conversion/reversal/commission settlement, no Cartasy order cho partner purchase.
- [ ] `F3-REC` Personalization experiment/guardrail/opt-out; provider/queue/search engine chỉ theo số đo và ADR.

Không đánh dấu F3 monetization xong vì click/impression hoặc mock payment success. Bằng chứng cần hợp đồng/chứng từ/đối soát và test nguồn dữ liệu khác nhau theo [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

## Definition of Done

Mỗi change có domain owner + business rule/state/permission + schema/migration + API/event contract + app UX + failure/retry/observability + test/race/security phù hợp + cập nhật tài liệu sở hữu. Checklist không được đánh dấu xong bởi sơ đồ/scaffold/placeholder.
