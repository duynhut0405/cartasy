---
doc_id: DOMAIN-RECOMMENDATIONS
title: Recommendations and Ranking
status: accepted
owner: Discovery
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Recommendation F2/F3, nguồn dữ liệu, ranking và fairness
---

# Gợi ý sản phẩm (F2/F3)

F2 bắt đầu bằng rule-based: cùng category/brand, sản phẩm liên quan đã duyệt, trending từ aggregate không định danh; chỉ listing/shop active/available, loại product đang xem, duplicate và tài nguyên buyer không được xem. Mọi card gắn shop/giá listing hợp lệ và có fallback deterministic nếu projection chậm. Không dùng tổng “đã bán” mô phỏng làm social proof thật. F3 cá nhân hóa chỉ sau consent rõ, data minimization, retention và đánh giá privacy/abuse; user tắt personalization sẽ về rule-based. Recommendation không tính lại giá/tồn kho cho checkout.

Ranking pipeline: candidate generation → eligibility filter → policy/safety → score → diversity/dedupe → sponsored placement **tách nhãn** F3. Organic score versioned, admin có khả năng audit vì sao item được hiện, không cho seller tự ghi ranking. Sponsored item phải đáp ứng cùng eligibility và disclosure, không len vào sort giá/relevance như kết quả hữu cơ. Experiment A/B cần assignment ổn định, guardrail conversion/latency/complaint và kill switch; không tự tuyên bố “AI” nếu chỉ là rule.

API `GET /v1/recommendations?context=home|product|cart&referenceId=...` trả IDs/listings với modelVersion/asOf, giới hạn 1–30; profile cá nhân không lộ qua cache công khai. Projection nhận event versioned/consent-aware; remove/takedown/suspend có hiệu lực ngay qua eligibility primary/public projection guard. Lỗi projection trả fallback (không chặn page), bad context `422`. Test consent off, cold start, shop suspension, duplicate, fairness/diversity, sponsored label và không làm sai giá checkout.
