---
doc_id: DOMAIN-INDEX
title: Bản đồ module nghiệp vụ
status: accepted
owner: Product & Engineering
last_updated: 2026-10-19
version: 4.1.0
source_of_truth: Quyền sở hữu và quan hệ giữa các đặc tả domain
---

# Module nghiệp vụ

Mỗi file sở hữu state machine, validation, policy, lỗi nghiệp vụ và sự kiện của module. HTTP shape/path chuẩn nằm ở [API contracts](../system/api-contracts.md); schema/constraint chuẩn nằm ở [data model](../system/data-model.md). Module khác chỉ gọi public application API/port của owner.

| Context | File | Giai đoạn |
|---|---|---|
| Identity & Customer | [Identity, profile, session](identity-profile.md) | F1 |
| Communications | [Notification, email, OTP](communications.md) | F1 |
| Commerce catalog | [Product, category, listing, media](catalog.md) | F1 |
| Discovery | [Search, filter, sort](discovery.md) | F1 |
| Customer intent | [Wishlist](wishlist.md) | F1 |
| Cart & checkout | [Cart, quote, checkout](cart-checkout.md) | F1 |
| Payments | [Payment simulation](payment-simulation.md) | F1 |
| Orders | [Order & order line](orders.md) | F1 |
| Fulfillment | [Shipping simulation](shipping-simulation.md) | F1 |
| Inventory | [Stock & reservation](inventory.md) | F1 |
| Merchandising | [Promotion & voucher](promotions-vouchers.md) | F1 |
| Trust | [Review & moderation](reviews.md) | F1 |
| Trust & Safety | [Abuse report, takedown & appeal](risk-moderation.md) | F1 |
| After-sales | [Return & refund simulation](returns-refunds.md) | F1 |
| Service | [Customer support](customer-support.md) | F1 |
| Marketplace | [Shop & seller onboarding](marketplace-seller.md) | F1 |
| Content | [CMS & merchandising](cms-content.md) | F1 |
| Reporting | [Shop & platform reporting](reporting.md) | F1 |
| Growth | [Flash sale](flash-sale.md) | F2 |
| Discovery | [Recommendations](recommendations.md) | F2/F3 |
| Growth | [Outbound affiliate và partner revenue](affiliate.md) | F3 có điều kiện |
| Monetization | [Advertising và sponsored placement](advertising.md) | F3 có điều kiện |
| Monetization & Finance | [Billing quảng cáo thật](advertising-billing.md) | Sau MVP, chỉ khi đủ gate |

`F1` là marketplace đa seller hoạt động với giao dịch/giao vận mô phỏng. `F2` thêm flash sale/recommendation/bulk tooling nếu được duyệt; `F3` mở quảng cáo/affiliate theo gate. Tiền quảng cáo/hoa hồng đối tác thật ở tương lai **không** đi qua buyer order/payment mô phỏng. Quy tắc shop, snapshot và tenant bắt buộc từ F1. [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

Mỗi đặc tả trả lời: vai trò/điều kiện truy cập, hành vi và trạng thái, input validation, transaction/event, lỗi và phản ứng UI, API và màn hình liên quan, test bắt buộc. App không được suy ra quyền từ UI; API kiểm tra policy trên resource thật. Dữ liệu mô phỏng phải được gắn nhãn rõ ở checkout, payment, shipment, refund và review liên quan tới trải nghiệm giả lập.
