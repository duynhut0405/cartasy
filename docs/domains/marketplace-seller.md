---
doc_id: DOMAIN-MARKETPLACE
title: Marketplace Shop and Seller Lifecycle
status: accepted
owner: Marketplace
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Shop onboarding, tenancy, membership, seller permissions
---

# Marketplace đa seller (MVP)

## 1. Aggregate và trust boundary

`Shop` là tenant thương mại; một identity có thể là buyer và member nhiều shop. Mỗi listing, stock, shop order, promotion, support ticket shop-scoped và seller report gắn `shop_id` từ MVP. API lấy actor từ session, kiểm membership active + capability + shop status + owner của resource **trên mỗi request**; `shopId` trong path/body không cấp quyền. Không có seller-specific user table/credential tách Better Auth. Admin quản trị platform không mặc nhiên được hành động thay seller mà thiếu capability/audit reason.

## 2. Đăng ký và kích hoạt shop

Buyer email verified + recent auth + MFA setup → `application.draft` → `submitted` → `under_review` → `approved/rejected/needs_changes`. Đơn gồm tên shop 3–120 ký tự, slug đề xuất cho bản `vi` unique theo locale, mô tả <=2000, liên hệ và hồ sơ xác minh tối thiểu; media đã scan. F1 không thu số tài khoản ngân hàng/KYC tài chính vì không payout cho mock order; advertiser billing sau này có onboarding tài chính riêng. Admin reviewer không duyệt application của chính mình; quyết định có reason/audit, nếu `needs_changes` seller sửa version và nộp lại. Approval tạo shop `draft` + owner membership trong transaction, outbox notification; activation là command riêng của owner sau khi profile/media/Terms version hiện hành và capability đủ điều kiện, kiểm lại trên server, ghi audit/event đúng một lần. Shop `draft → active → suspended → closed`; suspended ẩn listing, chặn checkout mới và seller catalog mutation, giữ lịch sử/order/support. Closure giữ snapshot/order/audit, xử lý yêu cầu tồn đọng theo policy. Ai vận hành đơn cũ sau suspend là [Open Decision M-01](../delivery/audit-and-open-decisions.md), không tự hủy.

## 3. Membership và capability

Role template `owner/manager/staff/analyst`; capability granular: `shop.profile.manage`, `listing.manage`, `inventory.manage`, `orders.manage`, `promotions.manage`, `support.reply`, `returns.review`, `reviews.reply`, `analytics.view`, `members.manage`. Shop owner không thể tự cấp `payment.outcome` hay quyền platform. Invite email verified, token hash một lần/TTL 48 giờ, shop ID/role/capabilities snapshot; invitee chấp nhận qua command riêng sau login+MFA, token one-use/actor email match/expiry/role guard trong transaction. Duplicate/revoked/expired trả conflict; không tạo membership chỉ vì email được gửi. Revoke và role change có hiệu lực request kế tiếp; session cache phải invalidate. Owner cuối cùng không thể tự rời shop khi chưa chuyển quyền; chuyển owner là command có recent auth/MFA cho cả hai bên và audit. Suspend membership dừng access ngay. API trả 404 cross-shop resource, 403 capability, 423 shop suspended, 409 stale version/state.

## 4. Khu Seller trong Admin workspace và vận hành

Khu `/seller/*` của **cùng `apps/admin`** cung cấp shop switcher, onboarding, profile, product proposal/listing/price, stock adjustment, promotion cơ bản, shop orders, return/support scoped và reporting aggregate. Mỗi tab/request/query key có workspace kind và shop ID; đổi shop/workspace cancel request/stream, clear cache/form. Seller không sửa canonical product của shop khác, không đổi mock payment result, không thấy buyer địa chỉ đầy đủ khi không có giao hàng thật. Product proposal qua moderation; listing `draft → pending_review → active ↔ paused → archived`, Platform Admin duyệt publish lần đầu; chỉnh giá/stock sau publish theo capability và audit/version. Seller shop-order transition theo [Orders](orders.md)/[Shipping](shipping-simulation.md), không được skip state.

## 5. Shop public và nhiều shop một order

Web shop page chỉ hiện shop active, thông tin đã duyệt, listing eligible; không hiện “verified” nếu chỉ duyệt nội bộ mà chưa có chương trình verified. Buyer cart có listing nhiều shop; checkout split thành N shop order, phí/voucher/snapshot theo shop. Seller chỉ nhận notification/báo cáo phần của mình; buyer xem parent đầy đủ. Shop suspend sau khi buyer xem quote khiến checkout `SHOP_SUSPENDED` và phải xác nhận lại; đơn đã tạo giữ snapshot, admin xử lý fulfillment mô phỏng còn dở.

## 6. API/data/event/test

API `/v1/seller/applications`, `/v1/seller/shops`, `/v1/seller/shops/:shopId/*` và `/v1/admin/shop-applications/*` theo [API contract](../system/api-contracts.md). Bảng `shops, shop_applications, shop_memberships, shop_invitations, shop_capabilities`; unique localized slug/membership, version, FK và audit ở [Data model](../system/data-model.md). Event `shop.application_submitted/approved/rejected`, `shop.activated/suspended`, `shop.membership_changed` sau commit; worker phân phối đúng recipient.

Test tối thiểu hai seller/hai shop/hai buyer và một identity có cả seller/platform role: cross-tenant read/write/export/notification/analytics bị từ chối, role revoke trong phiên, invitation replay, last owner, suspended shop quote/checkout, product proposal moderation, multi-shop order/refund và cache/workspace switch. [Admin workspace guide](../apps/admin/README.md) sở hữu `src`/setup; [Security](../system/security-privacy.md) sở hữu RBAC/ABAC matrix.
