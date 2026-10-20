---
name: dev-be
description: "Triển khai backend Cartasy khi thay đổi nghiệp vụ, PostgreSQL, Nest API, authz hoặc OpenAPI. Áp dụng cho packages/server, packages/database và apps/api."
---

# Dev BE

- Đọc domain spec sở hữu, `docs/system/data-model.md`, `docs/system/api-contracts.md`, `docs/system/security-privacy.md` và app guide API trước khi code. Khi đổi rule, cập nhật đặc tả và contract cùng thay đổi.
- Đặt rule/use case/port trong `packages/server`, repository và transaction adapter trong `packages/database`; `apps/api` chỉ xử lý HTTP/auth/composition. Không deep import qua module owner.
- Server quyết định VND integer, giá, tồn kho, phí và trạng thái. Mọi shop resource lấy scope `shop_id` từ ngữ cảnh đã xác thực; kiểm capability và resource policy tại API/application layer.
- Checkout atomic, idempotent; Cartasy payment/shipping/refund chỉ mô phỏng. Đồng bộ migration, DTO/OpenAPI, generated client và test PostgreSQL/HTTP/contract trước khi bàn giao UI.
