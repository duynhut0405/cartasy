---
doc_id: DOMAIN-IDENTITY
title: Identity, authorization, profile và session
status: accepted
owner: Identity & Customer
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: Hành vi identity/customer; ma trận quyền toàn hệ thống ở security-privacy
---

# Identity, profile và session

## Actor và vòng đời

Buyer có thể xem catalog không đăng nhập; cart server, checkout, wishlist đồng bộ, review, hỗ trợ và lịch sử đơn cần session. Seller/admin là cùng identity nhưng quyền theo membership/role; không tạo bảng tài khoản thứ hai. Đăng ký F1 bằng email/password → `pending_verification` → email xác minh hợp lệ → `active`. Tài khoản bị `suspended` không được tạo session mới và session hiện hành bị thu hồi. Xóa tài khoản theo yêu cầu đi qua trạng thái `deletion_requested`, ẩn danh dữ liệu có thể xóa nhưng giữ snapshot đơn/nhật ký theo retention.

Better Auth sở hữu credential, verification, reset token và session. `customers` sở hữu tên hiển thị, số điện thoại tùy chọn, locale và address book. Email đổi phải xác minh địa chỉ mới trước khi có hiệu lực; gửi thông báo đến địa chỉ cũ. Mọi thao tác nhạy cảm (đổi email/password, khóa tài khoản, thêm admin, mở shop) yêu cầu recent authentication; admin cần MFA.

## Quy tắc phiên và OTP

- Session cookie `HttpOnly`, `Secure` ở HTTPS, `SameSite=Lax`, host-only tại API origin; CORS origin allowlist và CSRF/Origin check cho mutation. Session được rotate khi đăng nhập, nâng quyền, reset credential và thu hồi khi logout/password reset/suspension.
- Phiên buyer: idle 14 ngày, absolute 30 ngày; admin/seller: idle 8 giờ, absolute 24 giờ. Mọi request kiểm tra revoke/expiry; không suy quyền từ claim cũ sau khi membership thay đổi.
- Email verification/password reset dùng token một lần, random, hash ở DB, TTL 15 phút và 30 phút tương ứng; resend tối đa 3 lần/giờ/email/IP. Phản hồi reset luôn trung tính để tránh enumeration. TOTP là MFA của admin/seller; recovery code một lần, lưu hash. OTP email không được coi là xác nhận thanh toán.
- Giao diện login hiển thị lỗi chung; API rate limit theo IP và account, có backoff. Log không chứa password, raw token, session cookie hoặc mã OTP.

## Profile và địa chỉ

Tên hiển thị 2–80 ký tự sau trim; phone theo E.164 nếu có; preferred UI locale `vi` hoặc `en` (mặc định `vi`; formatter `vi-VN`/`en-US`, tiền vẫn VND), timezone hiển thị từ profile. Địa chỉ gồm recipient name, phone, province/district/ward, detail; server validate độ dài, mã vùng theo danh mục versioned và strip control character. F1 địa chỉ là dữ liệu cho **mô phỏng**, UI nói rõ không giao hàng. Một buyer có tối đa 20 địa chỉ active, một mặc định; đổi/xóa địa chỉ không thay snapshot trên đơn đã đặt. Profile update dùng `If-Match`/version để tránh ghi đè.

## Quyền, lỗi, tích hợp

`GET/PATCH /v1/me` và `/v1/me/addresses` chỉ truy cập actor hiện tại; id lạ trả `404`. `/v1/auth/*` theo provider adapter đã pin. `401` khi session mất/hết hạn, `403` khi account suspended/thiếu capability, `409` khi version khác, `422` khi dữ liệu sai, `429` khi rate limit. Web chuyển người dùng đến login rồi trở lại checkout với cart còn nguyên; mutation sau login phải lấy lại quote. Admin/seller không được dùng vai trò buyer để vượt shop policy. Ma trận role/resource chuẩn tại [Security](../system/security-privacy.md).

Sự kiện `identity.email_verified`, `identity.session_revoked`, `customer.profile_changed` phát qua outbox khi domain cần downstream; không gửi email đồng bộ trong transaction. Test: replay token, concurrent default address, cross-user ID, session revoke, MFA/role change, CORS/CSRF, reset enumeration.


## Contract triển khai MVP

| Use case | API chủ đạo | Ghi/đọc | Quyền và lỗi |
|---|---|---|---|
| Sign-up/verify/sign-in/recovery | `/v1/auth/*` Better Auth snapshot | Auth user/session/verification, email adapter | Rate limit; token expired/replayed; response chống enumeration |
| Profile/address | `GET/PATCH /v1/me`, `/v1/me/addresses` | `customer_profiles/addresses` | Owner-only, `If-Match`, cross-user 404 |
| Seller/admin elevation | Better Auth MFA + `/v1/seller/shops`/`/v1/admin/session` | membership/role/version | Recent auth + TOTP; revoke hiệu lực request tiếp theo |
| Account deletion/export | `POST /v1/me/data-exports`, `GET /v1/me/data-exports/:id`, `POST /v1/me/deletion-requests`, `GET /v1/me/deletion-requests/:id` | request/audit/job | Reauth, retention/hold, không hard-delete order snapshot |

Session idle/absolute tính từ server clock, touch không được vượt absolute; rotate khi nâng quyền và thu hồi mọi thiết bị theo policy reset. Address/order snapshot liên kết ID nhưng dữ liệu sau checkout bất biến. Identity không tự gửi OTP bằng client; provider token one-use hash, email qua Communications. Test ma trận browser Web/Admin cùng API cookie, Seller↔Platform workspace switch trong Admin, logout một app và state app còn lại, stale membership, parallel reset/verify và account export/delete PII.

Admin user operation F1 tối thiểu: list/detail field allowlist; suspend/reactivate có `users.manage`, reason, `If-Match`, MFA/recent auth và audit; suspend thu hồi session, chặn login mới nhưng giữ order/support/audit snapshot. Platform role grant/revoke cần `roles.grant`, không tự nâng quyền hoặc gỡ System Admin cuối cùng; session/quyền đổi có hiệu lực request tiếp theo. Bulk user import, automated ban và policy appeal cuối cùng còn mở, không suy từ route Admin. [API contracts § 9](../system/api-contracts.md), [Security § 3](../system/security-privacy.md).
