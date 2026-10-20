---
name: dev-admin
description: "Xây workspace Seller và Platform trong apps/admin, gồm auth/MFA UX, capability-aware UI, shop switching và thao tác quản trị. Không dùng cho apps/web."
---

# Dev Admin

- Đọc `docs/apps/admin/README.md`, `docs/system/security-privacy.md`, domain spec và API contract của tác vụ.
- Giữ Seller `/seller/*` và Platform `/platform/*` trong một app. Menu theo capability chỉ định hướng UX; API kiểm role, shop membership, trạng thái tài nguyên và MFA/recent auth ở từng request.
- Khi đổi actor/workspace/shop hoặc logout, xóa React Query cache liên quan, form draft và stream/subscription. Không chia sẻ query key giữa hai workspace.
- Dùng shadcn/ui Radix trong Admin; mutation có `If-Match`/audit reason khi contract yêu cầu. Kiểm denied state, cross-shop và deep link với ít nhất hai shop, hai vai trò.
