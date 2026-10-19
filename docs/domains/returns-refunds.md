---
doc_id: DOMAIN-RETURNS
title: Return và refund simulation
status: accepted
owner: After-sales
last_updated: 2026-10-19
version: 4.2.0
source_of_truth: Yêu cầu đổi trả, quyết định và hoàn tiền mô phỏng
---

# Đổi trả và hoàn tiền mô phỏng

## Phạm vi, actor và quyền

Không có kiện hàng hay chuyển tiền thật. Buyer yêu cầu trả từng order item thuộc mình trong 168 giờ từ `delivered_simulated` (mặc định kỹ thuật cần Product duyệt theo [audit](../delivery/audit-and-open-decisions.md)); seller với `returns.review` chỉ đọc/đề xuất xử lý item của shop mình; admin `returns.decide` phê duyệt hoặc từ chối cuối cùng trong F1, tách quyền khỏi người tạo proposal. Tất cả màn hình, email, API và báo cáo ghi rõ “hoàn tiền mô phỏng”. Không thu tài khoản ngân hàng hoặc địa chỉ gửi trả.

## State, validation và số tiền

Request `requested → under_review → approved/rejected`, approved → `refunded_simulated` sau khi refund ledger ghi thành công; rejected terminal, buyer có thể mở ticket. Buyer chọn item, quantity >0 không vượt phần chưa có request pending/approved, reason `changed_mind/item_issue/other`, mô tả tối đa 2000 ký tự và tối đa năm ảnh đã scan. Request có `shop_order_id` duy nhất: một request không trộn item của hai shop. Deadline tính từ event giao đã lưu, không từ đồng hồ client; event giao sau điều chỉnh không tự kéo dài cửa sổ.

Server khóa order item/return allocation và payment ledger khi quyết định; tổng quantity pending+approved không vượt ordered quantity. Refund = snapshot unit price sau discount phân bổ × quantity + phí ship theo policy. F1 chỉ hoàn ship khi toàn shop order được hoàn do lỗi hệ thống/shop; cùng khoản ship chỉ được phân bổ một lần. `refunded_minor <= paid_minor`; refund unique theo return decision, retry idempotent. Stock mặc định không restock vì không có hàng thật; nếu admin chọn restock phải có reason và inventory movement đúng một lần. Parent/shop order giữ trạng thái giao nếu hoàn một phần, thêm summary; toàn phần chuyển `returned_simulated` theo [Orders](orders.md).

## API, event và lỗi

Buyer `GET/POST /v1/me/returns` và `GET /v1/me/returns/:id`; seller `GET /v1/seller/shops/:shopId/returns`, `POST /v1/seller/shops/:shopId/returns/:id/proposal`; admin `GET /v1/admin/returns`, `POST /v1/admin/returns/:id/decision`. Mutation quyết định dùng `If-Match`, idempotency và reason. Cross-owner/shop trả `404`; quá hạn `409 RETURN_WINDOW_CLOSED`; quantity `409 RETURN_QUANTITY_EXCEEDED`; state/version `409`; input/ảnh `422`. [API contracts](../system/api-contracts.md) sở hữu envelope; [data model](../system/data-model.md) sở hữu `return_requests, return_items, payment_refunds`.

Quyết định, refund, order summary và outbox ghi trong cùng transaction; `return.requested/decided/refunded_simulated` cho inbox buyer/seller/admin và báo cáo. Worker retry không tạo refund hoặc notification trùng. Payment ledger là owner số tiền, Return chỉ yêu cầu use case refund của Payment.

## UI và kiểm thử

Web cho thấy thời hạn/quantity còn lại, ước tính hoàn và giải thích không gửi hàng/không có tiền chuyển khoản; khu Seller `/seller/*` trong Admin workspace chỉ thấy đơn của shop; khu Platform `/platform/*` có evidence, proposal và audit theo capability. Test multi-shop isolation, partial/full, hai request cạnh tranh, deadline, refund cap, retry decision, shipping fee phân bổ, stock movement, seller không tự phê duyệt và email/notification dedupe.
