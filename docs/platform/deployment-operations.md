---
doc_id: OPS-RUNBOOK
title: Cartasy Deployment and Operations
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 4.2.0
source_of_truth: Môi trường, release, migration, backup, observability và incident runbook
review_cycle: Trước release và khi đổi hạ tầng
---

# Triển khai và vận hành

## 1. Topology chuẩn

Baseline triển khai **bốn** Node.js process/container (`web`, `admin`, `api`, `worker`), PostgreSQL managed, private S3-compatible bucket và SMTP provider. Một Admin origin phục vụ cả Seller/Platform; reverse proxy/TLS phục vụ ba origin Web/Admin/API là các subdomain cùng registrable domain đã sở hữu để cookie API host-only hoạt động trong same-site request; kiểm tra CORS credential và auth trên HTTPS. Không phụ thuộc Railway, Cloudflare hoặc DigitalOcean trong đặc tả; chọn provider phải chứng minh Node runtime, PostgreSQL backup/PITR, object storage private, secret manager, health check và môi trường tách biệt. Staging và production tách database, bucket, secret, domain và credential.

Local dùng Docker Compose cho PostgreSQL, Mailpit và object store; app chạy host/container. `APP_ENV` kiểm soát policy, không chỉ nhãn log. Production vẫn dùng mock payment và không có real fulfillment cho Cartasy order. Ad billing/affiliate outbound F3 không là thay `MOCK_PAYMENT_MODE`: dùng provider/partner adapter, secret, namespace, monitoring, audit và kill switch riêng sau contract/gate. [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

## 2. Environment và secret

| Môi trường | Mục đích | Gate |
|---|---|---|
| Local | Phát triển | Seed demo, scenario control test-only |
| Test/CI | Unit/integration/contract | Database/object store cô lập, data tự dọn |
| Staging | Gần production | HTTPS, email test provider, mock payment, migration/restore/smoke |
| Production simulation | Người dùng thật trong sản phẩm mô phỏng | Approval, privacy/security/recovery gate |

Typed config ở [Tech Stack](stack-dependencies.md); fail-fast nếu biến bắt buộc thiếu hoặc host/origin không hợp lệ. Secret runtime trong secret manager; CI deploy credential tối thiểu. Không in App spec/env vào log hoặc artifact. Client bundle chỉ có allowlisted public URL/config. Scenario override của mock payment phải bị từ chối ngoài local/test ở cả build và runtime.

## 3. CI và release

Pull request chạy frozen install, format/lint, dependency boundary, typecheck, unit/integration/contract, migration clean+upgrade, build, OpenAPI drift, secret/dependency scan và documentation link check. Workflow quyền tối thiểu, action pin commit SHA, không cấp deploy secret cho untrusted PR.

Trình tự staging/production:

1. Chốt commit/release ID, changelog, migration impact, feature flags và rollback plan.
2. Backup/check restore point, kiểm tra capacity/connection budget và secret.
3. Chạy migration job **một lần** có advisory lock; migration tương thích phiên bản app N/N-1.
4. Rollout API/worker theo dependency, sau đó Web/Admin chung; health/readiness không nhận traffic khi schema/dependency chưa sẵn sàng.
5. Smoke auth, catalog/search, cart, mock success/failure/timeout, order/notification (inbox/email/Web Push/SSE/campaign), seller onboarding/two-shop isolation, admin/tenant policy, headers/CORS.
6. Quan sát error/latency/outbox/stock reconciliation trong monitoring window; lưu commit, artifact, operator, thời gian và kết quả.
7. Production cần approval riêng sau staging xanh; không auto-deploy chỉ vì merge.

Ứng dụng rollback về image trước nếu schema còn tương thích. Migration dữ liệu đã chạy dùng forward fix khi rollback gây mất order/stock. Không chạy `drizzle-kit push` hoặc sửa migration lịch sử trên shared environment.

## 4. Backup và restore

PostgreSQL: backup managed/PITR và logical dump được mã hóa vào nơi tách biệt; object store versioning/backup cho media; secret được khôi phục theo runbook riêng. Mục tiêu thiết kế trước public launch: RPO ≤15 phút, RTO ≤2 giờ cho dữ liệu order/payment/stock, chỉ khi provider và drill thực tế đạt. Retention backup mặc định 30 ngày, quyền restore hạn chế và audit. Staging restore drill vào database/container **khác nguồn**, so schema, row count, tổng order/shop order/payment/reservation và sample snapshot; ghi thời gian thực tế. Không dùng backup production chứa PII trong staging.

## 5. SLI/SLO ban đầu

Đây là mục tiêu để kiểm chứng, không phải kết quả đã đạt:

| SLI | Mục tiêu ban đầu |
|---|---|
| API availability | ≥99.5% theo tháng, loại maintenance đã công bố |
| Catalog/search p95 | ≤500 ms ở workload đã chốt |
| Checkout API p95 | ≤2 s tới order pending, không tính mock settlement |
| Mock settlement p95 | ≤30 s từ order pending |
| Notification delivery p95 | ≤60 s sau event nghiệp vụ |
| Data correctness | 0 order trùng theo key, 0 stock âm/oversell, 0 paid simulated thiếu attempt, 0 refund vượt paid |
| Restore | RPO/RTO đạt mục 4 trên drill |

Workload profile F1 cần tối thiểu 10k product, 100k listing, 50 concurrent buyer và burst checkout 10/s để tìm bottleneck; điều chỉnh target bằng dữ liệu usage thật trước public launch. Query plan và connection budget là một phần gate, không suy đoán scale từ số user đăng ký.

## 6. Observability

Mọi request có request ID; log JSON có route template/status/duration và redaction. Metric: API p50/p95/p99, 5xx, auth failure, checkout 409 theo code, stock reservation fail, mock payment pending age/failure, refund/order/shop order state mismatch, return pending/SLA overdue, support first-response/SLA overdue, moderation backlog, outbox backlog/oldest age/failed, email/push delivery, campaign fanout lag, SSE connections, PostgreSQL pool/lag/storage, object store error. Trace nối request → checkout transaction → outbox → mock settlement → notification bằng correlation ID, không ghi PII.

Alert có owner/on-call và kênh nhận thật trước production. Ngưỡng khởi đầu: readiness fail 2 lần liên tiếp; outbox oldest >5 phút; mock payment pending >2 phút; bất kỳ stock âm/order trùng/reconciliation mismatch; 5xx >2% trong 5 phút; backup/restore failure. Dashboard product analytics tách operational dashboard.

## 7. Runbook ngắn

| Sự cố | Hành động đầu tiên |
|---|---|
| API/database down | Dừng traffic ghi, kiểm tra readiness/pool/migration/release gần nhất; không chạy migration song song |
| Checkout conflict tăng | Phân loại `PRICE_CHANGED`/`OUT_OF_STOCK`/lock timeout, kiểm tra quote và stock; không sửa order thủ công |
| Mock payment pending lâu | Kiểm tra worker/outbox/attempt ID, replay idempotent, không tạo payment mới |
| Stock/order lệch | Tắt checkout feature flag, đối soát reservation/order/payment; chỉ adjustment có audit |
| Notification lỗi | Giữ event/delivery failed, sửa SMTP/Web Push/template rồi retry theo event ID; không rollback order |
| Campaign fanout mắc kẹt | Pause campaign, kiểm batch lease/checkpoint/provider 429/consent, replay idempotent; không gửi lại cho người đã nhận |
| SSE mất kết nối | Client reconnect bằng cursor và refetch inbox/order; kiểm connection cap/proxy timeout, không coi SSE là nguồn chuẩn |
| Catalog/media sai | Unpublish/disable listing và object public, bảo toàn provenance/audit |
| Secret lộ | Rotate/revoke, thu hồi session/key liên quan, quét log/artifact và post-incident review |

Severity, người trực, escalation và kênh thông báo phải có trong release record. Sau sự cố nghiêm trọng lưu timeline, impact, root cause, hành động phòng ngừa và owner.

## 8. Feature flag và kill switch

Có cờ tắt checkout, mock settlement worker, promotion, seller publish, ad placement và upload theo môi trường. Kill switch không làm mất order/outbox đang pending; khi bật lại, worker đối soát và replay idempotent. Flag thay đổi có audit/owner và thời hạn; không dùng flag để che route không được bảo vệ.

Khi triển khai monetization thật, thêm kill switch **riêng** cho ad serving, ad billing initiation/callback processing (không bỏ nhận receipt hợp lệ), từng affiliate partner/redirect và partner report import. Tắt redirect không xóa click/commission cũ; tắt ad billing không tự sửa invoice/ledger. Runbook có người trực Finance/Partnerships, đối soát provider/partner, dispute và thông báo cho advertiser. Không kích hoạt luồng tiền thật khi chưa có reconciliation và quyền vận hành được kiểm chứng.
