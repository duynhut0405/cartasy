---
doc_id: DOMAIN-AFFILIATE
title: Outbound Affiliate and Partner Revenue
status: accepted
owner: Marketplace Growth & Partnerships
last_updated: 2026-10-19
version: 4.1.0
source_of_truth: Outbound affiliate sang đối tác, attribution và hoa hồng thật có điều kiện
---

# Affiliate outbound — mua thật trên đối tác sau MVP

## Hai hành trình tách biệt

Buyer có thể (a) thêm `Listing` của shop Cartasy vào giỏ và đặt **đơn mô phỏng miễn phí**; hoặc (b) bấm CTA “Xem/mua trên [đối tác]” có nhãn liên kết tiếp thị và rời Cartasy để mua **thật trên đối tác**. CTA (b) không thêm vào cart, không áp voucher Cartasy, không tạo `orders/shop_orders/payments/shipments/returns` Cartasy và không hứa Cartasy xử lý thanh toán, giao vận hoặc hoàn tiền đối tác. Landing/đơn hàng thật thuộc Shopee/TikTok Shop/Lazada hoặc đối tác đã ký kết. [ADR-0010](../decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md), [Design](../system/design-system.md).

`ExternalOffer` là nội dung/sản phẩm đối tác được cấp phép, sở hữu `merchant_id`, partner product ID, destination allowlisted, source/provenance, rights expiry và thông tin giá/stock **nếu được đối tác cấp phép và có timestamp**. Nó không là `Listing`/SKU tồn kho Cartasy. Có thể liên kết editorial với `Product` qua mapping được duyệt, nhưng UI tách CTA và nguồn giá; nếu feed chậm/mất quyền thì ẩn giá/ảnh/CTA theo policy. Không scrape nội dung hoặc tạo affiliate link không được chương trình cho phép.

## Luồng và trạng thái

Partner program `candidate → due_diligence → approved → active → suspended/ended`. External offer `draft → pending_review → published → paused/expired`; URL/domain/creative và claim qua moderation. Click: buyer xem disclosure và chọn CTA → server kiểm offer/partner active, consent và destination allowlist → ghi `outbound_click` có opaque ID khi được phép → redirect an toàn tới URL do adapter tạo/đã duyệt. Không nhận URL redirect tùy ý từ query; chặn open redirect, SSRF ở ingestion, affiliate stuffing, self-click và bot. Nếu chưa được consent tracking, chính sách phải xác định redirect không tracking hoặc ẩn CTA; đây là Open Decision.

Conversion/commission **chỉ** ghi khi đối tác cung cấp report/API/callback hợp lệ, có source file/event ID, ký/xác thực theo contract, currency, thời điểm, reversal/refund và kỳ đối soát. Không suy conversion từ click, mock payment success, UTM hay việc Buyer quay lại Cartasy. State đề xuất `reported → validated → approved/payable → paid` hoặc `reversed/disputed`; tên state cuối phải khớp contract thực tế từng đối tác. `commission_estimate` khác `commission_approved` và `cash_received`; chỉ phần đã đối soát theo policy Finance mới được trình bày là doanh thu thật. Không tự hứa attribution window 7 ngày; theo điều khoản đối tác.

## Quyền, API và dữ liệu dự kiến

`affiliate_partners`, `external_offers`, `affiliate_links`, `outbound_clicks`, `partner_conversion_reports`, `affiliate_commissions`, `affiliate_reconciliations` là **blueprint F3**, không migration F1. Partner credential nằm trong secret manager, report thô được bảo vệ/retained theo hợp đồng; không gửi buyer PII/order Cartasy cho đối tác nếu không có căn cứ, consent và DPA. API F3 dự kiến public `GET /vN/external-offers/:id`, `GET /vN/affiliate/out/:opaqueCode` (302 sau guard); Admin quản lý partner/offer/report/đối soát; Seller chỉ xem offer/program của shop nếu có quyền phân phối và hợp đồng. Không cho Seller tự nhập bất kỳ affiliate URL rồi publish. Report Seller/Admin hiển thị click hợp lệ, conversion do đối tác xác nhận, commission theo trạng thái và `asOf`; không trộn với “giá trị đơn mô phỏng”.

## Partner due diligence và release gate

Trước khi chọn Shopee, TikTok Shop, Lazada hoặc đối tác khác: xác nhận Cartasy được chấp thuận tham gia **đúng chương trình và quốc gia**, website/app được phép đặt link, deep link/redirect, dùng tên/ảnh/giá, API/feed/report quyền nào, quota/rate limit thực tế, cookie/attribution window, chống gian lận, disclosure, tax/settlement và điều kiện dừng. Không suy quyền Affiliate từ quyền Seller/Open Platform nói chung. Các nguồn chính thức tra cứu ngày 19/10/2026: [Shopee Affiliate VN terms](https://shopee.vn/affiliate/dieu-kien-hop-tac-chuong-trinh-tiep-thi-lien-ket-cua-shopee-viet-nam-update-18-08-2023/), [Lazada Affiliate VN terms](https://pages.lazada.vn/wow/i/vn/corp/affiliate-terms), [TikTok Shop Affiliate Links — US](https://seller-us.tiktok.com/university/essay?knowledge_id=4049972027557678) (chỉ chứng minh cơ chế ở US, **không** chứng minh Cartasy đủ điều kiện ở Việt Nam). [Lazada Open Platform app limits](https://open.lazada.com/apps/doc/doc?docId=108055&nodeId=10433) là giới hạn app/API chung, **không** chứng minh quyền affiliate API. Điều khoản/limit thay đổi; xác minh lại trong tài khoản đối tác và lưu bản điều khoản áp dụng trước tích hợp.

Gate: link/domain allowlist, disclosure + consent, mobile deep link/browser fallback, expiry/rights, malformed/compromised URL, click dedupe/bot, report tamper/duplicate/out-of-order, reversal/refund, settlement reconciliation, privacy/export/delete, đối chiếu revenue và kill switch partner. Không triển khai callback giả làm nguồn doanh thu.
