# Hướng dẫn AI Agent — Cartasy

## Nguồn chuẩn

[docs/README.md](docs/README.md) là bản đồ Single Source of Truth 4.4.0. Đọc tài liệu sở hữu trước khi thiết kế/code; code hiện trạng chỉ là bằng chứng triển khai. Nếu code khác đặc tả, ghi chênh lệch và chuyển đổi có migration/test. Không biến đơn Cartasy mô phỏng thành giao dịch/giao hàng thật hoặc phục hồi credit/ví; quảng cáo có thể thu tiền thật và affiliate có thể dẫn mua thật trên đối tác **sau MVP**, qua boundary/gate riêng ở [ADR-0010](docs/decisions/ADR-0010-simulated-commerce-real-monetization-boundary.md).

## Thứ tự đọc và setup

1. [Scope](docs/system/product-scope.md), [feature map](docs/system/feature-map.md), [roadmap](docs/delivery/roadmap.md) và [business flows](docs/system/business-flows.md).
2. [Architecture](docs/system/architecture.md), [source architecture](docs/system/source-architecture.md), [stack](docs/platform/stack-dependencies.md), [data model](docs/system/data-model.md), [API contracts](docs/system/api-contracts.md), [security](docs/system/security-privacy.md).
3. [Domain spec](docs/domains/README.md) của vertical slice và [app guide](docs/apps/README.md) tương ứng.
4. [Local development](docs/platform/local-development.md), [migrations](docs/platform/database-migrations.md), [testing](docs/platform/testing-quality.md), [operations](docs/platform/deployment-operations.md), rồi [checklist](docs/delivery/build-checklist.md).

## Quy tắc thực thi

- Khởi tạo workspace/module boundary/dependency direction/schema/gate từ blueprint trước feature; không gom chức năng phát sinh vào `services` hay `utils` chung.
- Business rule nằm trong `packages/server`, persistence adapter ở `packages/database`, HTTP/UI/worker là adapter. Module chỉ gọi public API/port của owner.
- Giá VND integer, tồn kho, discount, phí, payment/order state do server quyết định. Client không gửi kết quả mock payment hoặc shop/user ID làm bằng chứng quyền.
- Checkout idempotent và atomic; payment/shipping/refund của **Cartasy order** đều mô phỏng, không thu tiền Buyer/giao hàng thật. Ad billing và affiliate partner revenue tương lai không dùng bảng/order/payment này. Outbox/worker at-least-once phải idempotent.
- Mọi tài nguyên shop có scope `shop_id` từ F1; role/capability/resource policy kiểm tra ở API/application layer. Seller và Platform Admin dùng chung `apps/admin` theo ADR-0011, chỉ có hai app frontend Web/Admin; không import database/server internals.
- Khi đổi nghiệp vụ, cập nhật domain spec → data/API contract → app guide/OpenAPI/migration/test → checklist trong cùng thay đổi. Chỉ đánh dấu hoàn thành khi có kết quả gate và bằng chứng chạy.
- Nếu quyết định mới ảnh hưởng lâu dài tới kiến trúc/phạm vi, tạo ADR. Giữ rõ nhãn mô phỏng trong UI, email và analytics.
