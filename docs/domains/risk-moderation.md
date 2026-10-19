---
doc_id: DOMAIN-RISK
title: Trust, abuse và moderation
status: accepted
owner: Trust & Safety
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Report, takedown, anti-abuse và appeal cho marketplace mô phỏng
---

# Trust, abuse và moderation

## Phạm vi F1

Marketplace đa seller cần đường xử lý nội dung và tài khoản vi phạm ngay trong MVP. Buyer/seller đăng nhập có thể report product/listing/shop/review vì spam, nội dung sai, xâm phạm quyền hoặc quấy rối; report không đồng nghĩa vi phạm. Admin moderator xem evidence, quyết định `no_action/content_hidden/listing_paused/shop_suspended/account_restricted` trong quyền được cấp, reason bắt buộc, audit và notification tới bên bị ảnh hưởng. Seller không moderation nội dung của mình; moderator không quyết định case có xung đột lợi ích. Appeal lần đầu qua support case liên kết, reviewer khác xem lại. Policy pháp lý chi tiết, thời hạn appeal và xử lý hàng cấm phải được chủ dự án duyệt trước public launch.

## State, dữ liệu và control

`report.open → triaged → investigating → resolved/dismissed`; duplicate report cùng actor/resource/reason trong cửa sổ chống spam được gộp nhưng giữ evidence/actor. Nội dung report không public. Action `proposed → active → revoked/expired`; active takedown ẩn search/listing/public CMS tức thì, chặn checkout mới, giữ order/return/support lịch sử. Shop suspension đi qua use case owner Shop; review hide qua Review; không ghi trực tiếp bảng domain khác. Abuse controls F1: rate limit auth/OTP/support/report, upload quarantine/scan, product/listing review trước publish, review moderation, checkout/idempotency abuse alert và audit trail. Không tự ban seller theo heuristic F1; automation F2 chỉ tạo case để người duyệt.

## API và quyền

`POST /v1/reports` cho authenticated actor với `resourceType/resourceId/reason/evidenceIds` allowlist; `GET /v1/me/reports` chỉ metadata case của actor. `GET /v1/admin/moderation/reports`, `POST /v1/admin/moderation/reports/:id/decision`, `POST /v1/admin/moderation/actions/:id/revoke` cho `moderation.manage` + MFA, `If-Match` và reason. Evidence URL private, signed, ngắn hạn; seller chỉ thấy outcome của resource mình và nội dung đã redacted. `404` cross-resource, `409` state/version/duplicate, `422` input, `429` spam. Bảng `abuse_reports, moderation_actions, moderation_appeals` thuộc Risk; policy action gọi owner module. [Security](../system/security-privacy.md) sở hữu kiểm soát chung; [API](../system/api-contracts.md) sở hữu envelope.

## Event, vận hành và test

`abuse.reported/triaged/action_applied/action_revoked` ghi outbox sau commit. Metric case backlog/age, false-positive/appeal overturn, abuse rates và takedown lag, không chứa PII ở label. Test report chéo shop, reporter privacy, moderator conflict, action rollback, checkout race với shop/listing bị ẩn, appeal reviewer tách biệt, spam/rate limit và evidence access. F2 có thể thêm detection/queue prioritization sau khi đo đủ false positive.
