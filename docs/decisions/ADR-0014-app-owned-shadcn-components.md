---
doc_id: ADR-0014
title: App-owned shadcn/ui components for Web and Admin
status: accepted
owner: Product & Engineering
last_updated: 2026-10-20
version: 1.0.0
source_of_truth: Vị trí source component UI của Web/Admin và điều kiện tạo component riêng
---

# ADR-0014 — Dùng shadcn/ui trực tiếp trong từng frontend

## Context

ADR-0013 dự kiến tạo `@cartasy/ui` chỉ để chia sẻ source shadcn/ui và token giữa Web và Admin. Hiện dự án có hai frontend với composition, routing và vòng đời UI khác nhau. Package UI riêng làm tăng cấu hình CLI, build và dependency boundary ngay từ F0 khi chưa có component riêng nào được chứng minh cần chia sẻ.

## Decision

- `apps/web` và `apps/admin` có `components.json` riêng. Chỉ thêm component shadcn/ui bản Radix thực sự được dùng; source sinh ra nằm trong `src/shared/ui` của app tương ứng và được review như code của dự án. Dependency UI được khai báo tại app dùng nó.
- Dùng component và khả năng tùy biến của shadcn/ui trước. Nếu thiếu element hoặc hành vi cần thiết, compose từ shadcn/Radix và semantic HTML trong feature hoặc `shared/ui` của app; chỉ tạo component riêng khi có use case cụ thể. Domain component như `ProductCard` hoặc `ListingSelector` ở feature/entity, không tạo một UI kit song song.
- Không tạo `packages/ui` hay `@cartasy/ui` trong F0/F1. [Design system](../system/design-system.md) sở hữu tên/giá trị token và accessibility contract; `globals.css` của mỗi app hiện thực cùng token cần dùng. Khi thay token chung, cập nhật cả hai app và kiểm visual/contrast.
- Chỉ xem xét package UI dùng chung về sau nếu đã có component/pattern cùng semantic contract được dùng ổn định ở cả hai app và chi phí duy trì bản sao thực tế lớn hơn chi phí package; khi đó cần ADR và migration riêng.

## Consequences

F0 chỉ cần hai `components.json`, không cần build/export package UI. Web/Admin có thể có bản source shadcn riêng để tùy biến theo nhu cầu; review tránh lệch token/accessibility ngoài ý muốn. Quyết định này thay **phần vị trí UI package** của [ADR-0013](ADR-0013-frontend-state-ui-tooling.md); quyết định về Zustand, Tailwind, Prettier và Husky trong ADR đó vẫn có hiệu lực.
