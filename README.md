# Cartasy

Web storefront F1 có route `/vi`/`/en` với `next-intl`; nội dung public được duyệt theo locale. [Cấu trúc Web](docs/apps/web/README.md).

Cartasy là sàn thương mại điện tử **đa nhà bán từ MVP** với một Admin workspace dùng chung cho Seller và nhân sự nền tảng theo RBAC/ABAC. Buyer mua sắm miễn phí; giỏ hàng, checkout, đơn hàng, thanh toán, vận chuyển và hoàn tiền trong Cartasy đều mô phỏng, không thu tiền Buyer hoặc giao hàng thật. Sau MVP, quảng cáo có thể thu tiền thật từ advertiser và affiliate có thể dẫn Buyer sang đối tác mua thật; hai luồng này tách khỏi đơn Cartasy. Không dùng credit/điểm để thanh toán mock order.

[Bộ tài liệu kỹ thuật chuẩn](docs/README.md) là nguồn thiết kế hiện hành để AI Agent setup, phát triển và bảo trì dự án. Bắt đầu từ [hướng dẫn AI Agent](AGENTS.md), [phạm vi sản phẩm](docs/system/product-scope.md), [kiến trúc](docs/system/architecture.md) và [hướng dẫn từng ứng dụng](docs/apps/README.md). Trạng thái triển khai thực tế được kiểm chứng bằng [build checklist](docs/delivery/build-checklist.md).
