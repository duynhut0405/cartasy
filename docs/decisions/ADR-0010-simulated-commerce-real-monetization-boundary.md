---
doc_id: ADR-0010
title: Simulated Commerce and Future Real Monetization Boundary
status: accepted
owner: Cartasy Product & Architecture
last_updated: 2026-10-19
version: 1.0.0
source_of_truth: Ranh giới giữa đơn mô phỏng Cartasy và doanh thu quảng cáo/affiliate có thể là tiền thật
review_cycle: Trước khi bật billing quảng cáo, tích hợp đối tác affiliate hoặc thay đổi checkout Cartasy
---

# ADR-0010: Tách giao dịch mua mô phỏng khỏi monetization thật

## Context

Cartasy cho Buyer mua sắm miễn phí qua checkout nội bộ với thanh toán, giao hàng và hoàn tiền mô phỏng. Định hướng kinh doanh mới xác nhận Cartasy có thể thu tiền thật từ quảng cáo dành cho Seller/Advertiser và nhận hoa hồng thật từ chương trình affiliate của đối tác; Buyer có thể rời Cartasy qua liên kết affiliate để mua thật **trên nền tảng đối tác**. Hai luồng này có chủ thể thanh toán, bằng chứng và nghĩa vụ khác nhau. [Product scope](../system/product-scope.md) sở hữu phạm vi; [Advertising](../domains/advertising.md) và [Affiliate](../domains/affiliate.md) sở hữu hành vi tương lai.

## Decision

1. `orders`, `shop_orders`, `payments`, `payment_refunds` và `shipments` của Cartasy chỉ ghi giao dịch **mô phỏng**. Buyer không trả tiền cho Cartasy để dùng luồng mua sắm mô phỏng. Không có trường/cờ hay adapter nào biến `paid_simulated` thành khoản thu thật.
2. Quảng cáo là một bounded context riêng. Nếu bật thanh toán quảng cáo thật sau MVP, `ad-billing` sở hữu invoice, receipt, credit note/refund và sổ đối soát quảng cáo; người trả tiền là advertiser/Seller theo hợp đồng. Không reuse `payments`/`orders` của Buyer; không cho kết quả quảng cáo quyết định trạng thái mock order.
3. Affiliate outbound là một luồng **rời Cartasy** có nhãn và sự đồng ý rõ trước khi chuyển tới đối tác allowlisted. Cartasy chỉ ghi click có consent và conversion/commission khi có bằng chứng hợp lệ từ đối tác; đơn mua thật thuộc đối tác, không tạo `shop_order` hay shipment trong Cartasy, không hứa hoàn tiền/hỗ trợ đơn thật thay đối tác.
4. Sponsored placement và affiliate link phải được nhận biết rõ, có kiểm duyệt nội dung/quyền sử dụng, chống redirect tùy ý và tách báo cáo `giá trị đơn mô phỏng` khỏi `doanh thu quảng cáo/hoa hồng đã đối soát`. Conversion từ mock checkout không chứng minh doanh thu thật.
5. F1 không tích hợp payment provider, ad billing hay partner purchase. F3 chỉ là **mục tiêu có điều kiện**, không bật bằng feature flag hoặc đổi provider khi chưa có hợp đồng, chính sách, điều kiện pháp lý/thuế, quyền dữ liệu, incident/reconciliation và gate kiểm thử được duyệt.

## Consequences

Kiến trúc có thể giữ một API/DB ở giai đoạn đầu nhưng bảng, module owner, API namespace, báo cáo và quyền tách biệt. Quảng cáo có thể hiển thị với sản phẩm Cartasy nếu được duyệt, nhưng thương mại quảng cáo không dùng giỏ hàng Buyer. External product/offer affiliate là tài nguyên khác `Product/Variant/Listing` và không được thêm vào Cartasy cart. Cần schema/event version và migration khi triển khai F3; không tạo bảng F3 ở F1. Cần nghiên cứu riêng điều khoản Shopee/TikTok Shop/Lazada theo quốc gia, chương trình và quyền API trước khi chọn đối tác.

## Revisit conditions

Nếu Product muốn Buyer thanh toán thật trong Cartasy, Seller payout, hoặc affiliate purchase đi qua Cartasy checkout, phải có quyết định sản phẩm/ADR mới và thiết kế lại ledger, pháp lý, bảo mật, hậu mãi. Không suy các luồng đó từ ADR này.
