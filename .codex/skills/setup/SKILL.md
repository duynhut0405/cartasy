---
name: setup
description: "Dựng hoặc sửa nền tảng F0 của Cartasy: workspace, app/package skeleton, môi trường local, migration và tooling. Dùng trước khi triển khai feature."
---

# Setup

- Đọc `docs/README.md`, `docs/system/source-architecture.md`, `docs/platform/stack-dependencies.md`, `docs/platform/local-development.md` và các mục S0–S3 trong `docs/delivery/frontend-first-setup-checklist.md`.
- Tạo bốn app Web/Admin/API/Worker và các package đúng blueprint. Giữ dependency direction và public exports; Web/Admin không import server hoặc database. Không tạo package UI chung.
- Pin version tương thích, dùng một lockfile; cấu hình env có validate. Dựng PostgreSQL, Mailpit, object store, migration/seed và health/readiness. Image pin + named volume chỉ giữ dữ liệu từng máy; máy mới phải dựng được từ migration/seed. Hỗ trợ DB/bucket dev riêng khi cần giữ cùng trạng thái qua nhiều máy.
- Chạy install, format/lint/typecheck/build và migration trên database sạch. Ghi kết quả thật vào checklist; không dùng mock backend làm nguồn dữ liệu runtime.
