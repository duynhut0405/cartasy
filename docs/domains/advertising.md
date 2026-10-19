---
doc_id: DOMAIN-ADVERTISING
title: Advertising and Sponsored Placement
status: accepted
owner: Marketplace Monetization
last_updated: 2026-10-19
version: 4.1.0
source_of_truth: Ad serving, disclosure và advertiser campaign sau MVP; billing thật thuộc module riêng
---

# Quảng cáo và sponsored placement — sau MVP

## Sản phẩm và ranh giới

Cartasy có thể bán Sponsored Product, Sponsored Shop, Search Ad, Display/Banner và fixed placement cho Seller/Advertiser sau MVP. Buyer vẫn mua sắm mô phỏng miễn phí; paid ad là dịch vụ riêng cho advertiser. `AdCampaign` sở hữu creative, placement, targeting, schedule, delivery cap và trạng thái; [Ad billing](advertising-billing.md) sở hữu hợp đồng/giá/invoice/receipt/ledger/refund **nếu thu tiền thật**. Không đặt số dư quảng cáo trong buyer wallet, không dùng `orders/payments` mô phỏng để thu phí. [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

## Placement và policy

Inventory placement đề xuất: home hero/collection, category list, search result, product detail related area và shop page khi phù hợp. Mỗi slot có ID, format, kích thước/responsive, giới hạn mật độ, eligibility và disclosure. Product/Shop ad phải dẫn tới resource active/published, qua cùng kiểm duyệt và guard stock/shop như organic; display ad có landing URL nội bộ hoặc domain đã duyệt. Sponsored card gắn `placement: sponsored` và nhãn “Quảng cáo” sát nội dung; không giả là kết quả organic, không chen bí mật vào sort giá/relevance, không dùng review/số bán hư cấu. Banner/creative không được ám chỉ Cartasy giao hàng hoặc thu tiền cho mock order.

Seller/advertiser tự tạo draft nếu có capability và shop/account hợp lệ; Admin duyệt creative/claim/URL/targeting, tách người duyệt đối với campaign rủi ro. State `draft → pending_review → approved → scheduled → active → paused/ended`, nhánh `rejected/cancelled`; sửa creative/targeting sau duyệt về pending review. Shop/listing suspend, quyền media hết hạn hoặc hợp đồng hết hiệu lực làm ad ngừng phân phối ngay, không chờ projection. Admin có kill switch từng slot/campaign và audit reason. Seller không thể tự gắn badge sponsored lên listing.

## Targeting, đo lường và ngân sách

F3 candidate targeting: query/category/context, vùng rộng nếu được consent và chính sách duyệt; không dùng địa chỉ giao nhận mô phỏng, PII hoặc dữ liệu nhạy cảm. Frequency cap theo actor/anonymous consent scope; từ chối personalization vẫn có thể thấy context ad nếu pháp lý/policy cho phép, không dùng tracking trái consent. Impression chỉ ghi khi viewability rule/version đạt; click dùng event ID, bot/self-click filtering, signed/referrer-safe redirect khi cần. Mock checkout completion có thể là **chỉ số tương tác mô phỏng**, không là doanh thu quảng cáo hay mua hàng thật.

Pricing model `CPC/CPM/fixed placement`, mức giá, ngân sách, overspend guard, make-good và billing trigger đều là **Open Decision**. Không được dùng event client chưa xác minh làm căn cứ invoice. Nếu chạy thử ads không thu tiền, UI/Finance ghi `trial/no-charge`, không gọi ngân sách đó là tiền đã nạp. Khi thu tiền thật, serving eligibility và spend counter phải dựa trên trạng thái commercial đã được [Ad billing](advertising-billing.md) xác nhận, idempotent theo event và có cap/reconciliation; refund/credit note không sửa lịch sử impression/click.

## Contract và gate

Schema tương lai: `ad_placements`, `advertiser_accounts`, `ad_campaigns`, `ad_creatives`, `ad_delivery_events`/aggregates; FK `shop_id` nếu advertiser là seller, `campaign_id`, policy/creative revision, uniqueness event ID. API namespace `/vN/advertiser/campaigns`, `/vN/admin/ads/*`, public discovery với trường sponsored/disclosure; wire DTO, actor matrix và OpenAPI viết khi Product chốt gói/pricing/slot. Báo cáo Seller chỉ aggregate của account/shop mình, Admin xem platform; không lộ buyer ID. Test slot disclosure, eligibility, moderation, consent, click fraud, event replay, cap concurrent, suspended shop, billing isolation và kết quả sort organic. Không mở route/schema F3 trong F1.
