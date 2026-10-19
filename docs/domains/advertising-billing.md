---
doc_id: DOMAIN-AD-BILLING
title: Future real advertising billing boundary
status: proposed
owner: Marketplace Monetization & Finance
last_updated: 2026-10-19
version: 0.1.0
source_of_truth: Ranh giới nghiệp vụ billing quảng cáo thật sau MVP; chính sách thương mại còn mở
---

# Billing quảng cáo thật — thiết kế có điều kiện sau MVP

## Ranh giới và điều kiện bật

Đây là luồng Seller/Advertiser **trả tiền thật để mua dịch vụ quảng cáo của Cartasy**, không phải Buyer trả tiền cho đơn mô phỏng. Chỉ triển khai khi Product/Finance/Legal chốt pháp nhân bán dịch vụ, đối tượng được mua, phương thức thanh toán, chính sách giá, thuế/hóa đơn, hoàn/credit note, tranh chấp, kỳ đối soát và provider. F1 không tạo route, bảng hoặc payment SDK này. [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md) cấm reuse buyer `payments/orders`.

## Aggregate và lifecycle đề xuất

`AdvertiserAccount` liên kết shop hoặc pháp nhân được duyệt; `AdOrder`/`AdContract` snapshot gói, giá VND, kỳ và điều khoản; `Invoice`/`PaymentReceipt`/`CreditNote`/`Refund` theo luật và provider được chọn; `AdLedgerEntry` append-only là sổ phụ đối soát, không trộn `payment_refunds` mô phỏng. Trạng thái hợp đồng dự kiến `draft → awaiting_payment → funded → active → completed`, nhánh `cancelled/disputed/refunded`; trạng thái cuối cùng và có cho chạy trước thanh toán hay không là **Open Decision**, không dùng như policy đã duyệt.

Ad serving chỉ tiêu ngân sách đã được xác nhận hợp lệ theo commercial policy; event impression/click là đo lường, không tự động thành khoản phải thu trừ khi pricing model và chống gian lận được duyệt. `CPC/CPM/fixed placement` là lựa chọn nghiên cứu; không chọn mặc định. Mọi điều chỉnh có reason, actor, chứng từ và audit; đối soát provider/bank ↔ receipt ↔ invoice ↔ ledger ↔ campaign spend; lệch thì chặn ghi nhận doanh thu và báo vận hành. Webhook/callback nếu có phải xác thực chữ ký, timestamp/replay, raw body, receipt unique, async idempotent; không nhận kết quả thanh toán từ browser.

## API, dữ liệu và quyền dự kiến

Namespace tương lai `/vN/advertiser/*` và `/vN/admin/ad-billing/*` tách `/v1/checkouts` và `/v1/orders`. Seller chỉ thấy account/shop mình và số liệu được phép; Finance Admin có capability `ad_billing.view/adjust/refund` tách `ads.manage`, MFA/recent auth, dual control cho refund/điều chỉnh lớn; không cho moderator nội dung tự sửa sổ tiền. Dữ liệu đề xuất ở [Data model](../system/data-model.md) chỉ là blueprint F3, không migration F1. Không lưu card PAN/CVV hoặc ngân hàng nhạy cảm nếu provider token hóa; retention, tax và privacy do review áp dụng chốt.

## Release gate

Trước khi mở thật: provider contract/sandbox, legal/tax review, immutable ledger và double-entry hoặc mô hình đối soát được Finance duyệt, idempotency/race/refund/dispute tests, invoice numbering policy, backup/restore, access segregation, audit, monitoring và reconciliation drill. Báo cáo phải tách `doanh thu quảng cáo đã đối soát` khỏi `giá trị đơn mô phỏng`; chỉ ghi nhận doanh thu theo policy kế toán được duyệt, không đồng nhất với số click hoặc tiền nạp. [Roadmap](../delivery/roadmap.md), [Testing](../platform/testing-quality.md).
