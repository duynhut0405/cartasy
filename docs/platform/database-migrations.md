---
doc_id: PLATFORM-DATABASE
title: Database schema và migrations
status: accepted
owner: Data Platform
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Quy trình schema, migration, seed và recovery
---

# Database và migrations

PostgreSQL 18 là nguồn chuẩn giao dịch; Drizzle schema trong `packages/database/src/schema/<context>.ts` mô tả type/constraint, migration SQL đã review trong `packages/database/src/migrations` là cách duy nhất thay shared DB. Danh mục entity/quan hệ/invariant chuẩn ở [data model](../system/data-model.md); mỗi context sở hữu bảng của mình. Repository adapter ánh xạ row ↔ domain, read model nhiều bảng chỉ đọc. Không cho controller/worker handler/frontend import client DB.

## Quy tắc schema

PK UUID/opaque; FK cho owner/resource, unique composite cho business key, CHECK cho trạng thái/số không âm, timestamp UTC, version integer cho optimistic concurrency. Tiền là integer minor VND trong DB, string ở JSON. `shop_id` có mặt ở listing/stock/shop order/promotion/ticket/seller operation từ F1. PII tách/scope, địa chỉ order snapshot bất biến trừ ẩn danh theo retention. JSONB chỉ cho snapshot có schema version, không thay bảng quan hệ/constraint. Index dựa trên query plan và fixture phân bố tương đương staging; cursor có tie-break ID.

## Migration workflow

1. Sửa schema owner và viết SQL migration; review FK, unique/check, index concurrently/lock impact, default/backfill, rollback/recovery. Không sửa migration đã chạy.
2. Dùng expand → backfill theo batch idempotent → switch code (đọc cũ/mới trong cửa sổ N/N-1) → contract sau khi mọi process đã nâng. Không deploy code phụ thuộc cột mới trước migration tương thích.
3. Runner lấy advisory lock, ghi migration history/checksum, chạy đúng một lần trước rollout; migration lỗi dừng deploy. Với DDL không transaction được, ghi checkpoint và runbook riêng.
4. CI tạo DB sạch, apply toàn bộ migration, seed fixture; tạo DB từ release trước, apply upgrade và verify dữ liệu/invariant; so Drizzle schema với SQL/applied state. Không dùng `drizzle-kit push` trên staging/production.
5. Seed reference data idempotent bằng natural key; demo buyer/admin/order chỉ local/test, tách khỏi production seed. Không copy dữ liệu thật sang local.
6. Backup/PITR và restore drill ở [operations](deployment-operations.md); trước destructive contract, xác nhận backup và version app compatible. Không xóa product/shop/listing có order history; archive và giữ snapshot.

Máy phát triển mới dựng baseline bằng migration đã commit và seed local/test idempotent từ cùng commit; seed dùng natural key/ID ổn định và media fixture có quyền sử dụng để chạy lặp không nhân bản dữ liệu. DB remote dev cá nhân chỉ seed lúc khởi tạo; sau đó dữ liệu phát triển nằm trong DB đó, không tự đồng bộ từ Docker volume của máy khác. Nếu cần chuyển đúng trạng thái local, dùng `pg_dump`/`pg_restore` vào DB rỗng, giữ migration history và chuyển object store cùng thời điểm; kiểm schema/row count sau restore. Quy trình môi trường và bảo vệ snapshot ở [local development](local-development.md).

## Giao dịch và đối soát

Checkout atomicity, reservation CAS, voucher hold, payment settlement và refund xem [architecture](../system/architecture.md) và domain files. Dùng transaction ngắn, lock theo thứ tự ID nhất quán; không gọi SMTP/S3 trong transaction. Outbox ghi cùng transaction nguồn; worker at-least-once + inbox dedupe. Reconciliation định kỳ: tổng order/shop order, paid/refunded cap, stock/reservation, coupon usage, outbox/delivery; sai lệch tạo alert/audit, không auto-heal tùy tiện.

Mọi migration cần test fixture upgrade, concurrent behavior cho bảng nóng và dữ liệu rollback/repair plan. Khi schema là F2/F3, chỉ tạo bảng khi feature triển khai; F1 đã có shop ownership core.

F1 locale migration: thêm translation tables theo owner và `translation_group_id` cho CMS bằng expand/backfill/switch/contract. Backfill slug/title/description/alt hiện có vào `vi` theo batch idempotent; phát hiện collision trước khi tạo unique index locale, đối chiếu số row, URL và published state. Không sinh `en` tự động. Đọc cũ/mới trong cửa sổ rollout, lưu slug redirect theo locale, bảo toàn order snapshot. Test DB sạch và upgrade từ fixture đơn ngữ, SEO/read path sau switch, rollback về bản `vi`. [ADR-0012](../decisions/ADR-0012-multilingual-storefront.md).

Khi thêm F3, migration `advertising`, `affiliate`, `ad-billing` theo owner tách biệt; không thêm cột “real” vào `payments`/`orders` mô phỏng để tái sử dụng checkout. `external_offers` không có FK bắt buộc tới Cartasy listing/cart; `ad_payment_receipts` có unique provider receipt và ledger append-only theo Finance policy. Migration/retention/reconciliation của tiền thật cần review riêng trước rollout. [Data model](../system/data-model.md), [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).
