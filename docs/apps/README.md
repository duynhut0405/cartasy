---
doc_id: APP-INDEX
title: Hướng dẫn triển khai theo ứng dụng
status: accepted
owner: Engineering
last_updated: 2026-10-19
version: 4.3.0
source_of_truth: App boundary và thứ tự setup
---

# Ứng dụng

Muốn xem toàn bộ thư viện, module `src` và nơi lưu state/store trước khi scaffold: [Preview source của bốn app và shared packages](source-preview.md). Bảng này tổng hợp tài liệu sở hữu bên dưới, không thay thế contract chi tiết.

| App | Trách nhiệm | Tài liệu triển khai |
|---|---|---|
| `apps/web` | Storefront public và tài khoản buyer với `/[locale]` (`vi`, `en`) | [Web Frontend](web/README.md) |
| `apps/admin` | **Một workspace** cho Seller và Platform Admin; route/capability tách theo actor | [Admin workspace](admin/README.md) |
| `apps/api` | HTTP API, auth adapter, DI composition | [Backend/API](api/README.md) |
| `apps/worker` | Outbox, thông báo, payment/shipping scheduler | [Worker](worker/README.md) |

F1 có **bốn app/process**: Web, Admin chung, API, Worker. Seller dùng `/seller/*` trong `apps/admin`; Platform Admin dùng `/platform/*`. Mỗi app có hướng dẫn dependency, `src`, routing, state, API, env, setup và gate. `packages/server` sở hữu nghiệp vụ; `packages/database` sở hữu adapter dữ liệu; app không tự viết business rule. [Source architecture](../system/source-architecture.md), [ADR-0011](../decisions/ADR-0011-unified-admin-seller-workspace.md).
