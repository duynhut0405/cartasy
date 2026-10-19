---
doc_id: DOMAIN-DISCOVERY
title: Search, filter và sort
status: accepted
owner: Discovery
last_updated: 2026-10-19
version: 4.1.0
source_of_truth: Quy tắc tìm kiếm, lọc và phân trang
---

# Search và discovery

F1 dùng PostgreSQL full-text/trigram trên projection public có thể rebuild từ product/category/listing/shop. Chỉ index product và listing đủ điều kiện public. Search không là nguồn giá, tồn kho hoặc quyền; add cart/checkout đọc authoritative data. Query chuẩn hóa Unicode, trim, giới hạn 120 ký tự, chặn wildcard/query abuse. Query rỗng trả collection mặc định; typo/prefix fallback có giới hạn chi phí. Thứ tự relevance dùng rank + tie-break `product_id`; sort khác: newest, price asc/desc, popular (derived), rating trải nghiệm mô phỏng (chỉ khi đủ mẫu, có nhãn, không diễn giải là chất lượng hàng thật). Price sort dùng giá listing eligible tối thiểu và phải hiển thị “từ”.

Filter allowlist: category/subtree, brand, shop, price range VND integer, availability, rating threshold; facet count tính trên tập đã lọc trừ facet đang xét để UI dễ hiểu. `minPrice <= maxPrice`, range hữu hạn, sort enum, limit 1–100. Cursor opaque gắn query/filter/sort/index-version, hết hạn hoặc thay filter trả `422` và UI reset trang. Không expose deep offset. URL query là nguồn state lọc của web để deep link/back/forward; input search debounce, request cũ cancel, loading/empty/error rõ ràng. SEO chỉ index canonical category/product và collection được duyệt; query tùy ý `noindex`.

Sự kiện catalog/listing/shop cập nhật projection sau commit, có retry/rebuild. Mục tiêu F1 projection lag p95 < 60 giây; admin thấy trạng thái indexing, alert khi lag > 5 phút. Không hứa “còn hàng” tuyệt đối từ search. **Shop filter thuộc F1** theo API hiện có; F2 có thể thêm ranking/seller tooling. F3 sponsored result phải gắn nhãn quảng cáo và không làm sai sort hữu cơ khi người dùng chọn giá. Autocomplete, search history và tìm không dấu có mức đặc tả khác nhau; xem [audit sản phẩm](../delivery/full-product-audit-2026-10-19.md) trước khi coi chúng là feature F1 bắt buộc.

Endpoint `GET /v1/products`, `/v1/categories`, `/v1/shops/:slug/products` theo [API contracts](../system/api-contracts.md). Test: Unicode tiếng Việt, ẩn unpublished, tenant isolation trong admin search, cursor ổn định, facet, index rebuild, performance trên dataset tương đương staging.


## Contract filter, ranking và fallback

| Input | Validation | Hành vi lỗi |
|---|---|---|
| `q` | Unicode normalize, trim, 0–120 ký tự, token count/complexity cap | 422 invalid; empty q về collection mặc định |
| `category/brand/shop` | Opaque ID hoặc slug hợp lệ, category subtree tối đa depth policy | Unknown public filter trả empty, không lộ draft |
| `minPrice/maxPrice` | VND integer string >=0, min<=max | 422 |
| `sort` | relevance/newest/price_asc/price_desc/popular/experience_rating | 422, không âm thầm đổi sort |
| `cursor/limit` | Cursor gắn filter/sort/index version, limit 1–100 | 422 nếu tamper/stale; client reset |

Search projection chỉ là read model; event catalog/listing/shop update có version và rebuild idempotent. Seller search trong Portal là route riêng có `shop_id` filter ở repository, không dùng public search để liệt kê draft của mình. Sponsored F3 trả placement/disclosure riêng, recommendation F2 đi qua [Recommendations](recommendations.md). Test PostgreSQL query plan ở dataset staging và fallback khi index lag > SLO; public result không bao giờ là bằng chứng còn hàng để checkout.
