---
doc_id: DOMAIN-PROMOTIONS
title: Promotion và voucher
status: accepted
owner: Promotions & Pricing
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Điều kiện giảm giá, phân bổ và lượt sử dụng
---

# Khuyến mãi và voucher

Campaign có scope platform/shop, loại `percent/fixed/free_shipping`, thời gian UTC, quota tổng/per-buyer, minimum subtotal, eligible category/product/listing/shop, max discount, priority và status `draft → scheduled → active → paused → ended`. Voucher là mã normalized không phân biệt hoa/thường gắn campaign; không lưu giá trị tiền ở client. Giá trị giảm VND integer, không âm, không vượt subtotal/fee áp dụng. Voucher shop không giảm item shop khác. F1 mặc định kỹ thuật cho một voucher code/order và promotion tự động theo priority (cần Product duyệt theo [audit](../delivery/audit-and-open-decisions.md)); không stack hai giảm giá cùng một item, free shipping chỉ giảm fee. Thay đổi stacking cần ADR và test phân bổ.

Quote kiểm tra eligibility nhưng không chiếm lượt. Checkout hold lượt trong transaction; payment success commit, failure/cancel release; return/refund theo chính sách campaign snapshot: mặc định không hoàn lượt sau đơn paid, trừ cancel toàn bộ trước shipping và campaign còn active. Partial refund dùng discount allocation trên từng order item, không tính lại theo rule hiện tại. Tranh chấp quota dùng atomic counter/unique redemption key; mỗi buyer/quota tính theo identity server.

Admin tạo/duyệt campaign, seller F1 chỉ campaign shop mình trong budget/policy được cấp. Public chỉ thấy promotion active; code invalid trả `COUPON_INELIGIBLE` với reason safe (expired, not eligible, exhausted) không lộ quota riêng tư. Web hiển thị subtotal, discount, shipping, total theo shop và toàn đơn, nhãn hết hạn; thay mã phải lấy quote mới. Sự kiện campaign change invalidates pricing/search badge; order snapshot bất biến. Test boundary thời gian/timezone, quota concurrency, item allocation rounding, multi-shop scope, cancel/refund và stale quote.


## Priority, quota và API

| Rule | Invariant |
|---|---|
| Scope | Platform promotion có thể áp nhiều shop; shop voucher chỉ item/phí shop đó |
| Stacking MVP | Một code/order; campaign auto theo priority, cùng item không nhận hai item-discount; shipping voucher riêng |
| Usage | Quote chỉ kiểm; checkout hold atomic, payment success commit, failure release; per-buyer key unique |
| Allocation | Discount integer theo item/shop, phần dư phân bổ stable ID; snapshot không tính lại khi refund |
| State | Draft/scheduled/active/paused/ended; seller shop campaign cần capability, Admin override có audit |

Seller F1 chỉ tạo campaign shop mình, không sửa platform campaign; Admin duyệt theo policy trước public. Voucher input normalize uppercase/trim, 1–32 ký tự allowlist; code không lộ quota cá nhân. Flash sale F2 dùng [Flash sale](flash-sale.md) có quota riêng, không thêm flag vào voucher cơ bản. API seller/admin/public eligibility theo [API contract](../system/api-contracts.md). Test hai shop coupon conflict, quota đồng thời, timezone/schedule, return policy, stale quote, shop suspend và negative total.
