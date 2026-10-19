---
doc_id: ADR-0001
title: Modular Monolith for Cartasy
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 2.0.0
source_of_truth: Quyết định topology nghiệp vụ
review_cycle: Khi có dữ liệu tải hoặc nhu cầu tách service
---

# ADR-0001: Modular monolith

## Context

Checkout phối hợp cart, stock, promotion, order và payment outcome với nhiều invariant phải nguyên tử. Đội ngũ giai đoạn đầu cần deploy/quan sát đơn giản và module có owner rõ.

## Decision

Một NestJS API modular monolith + worker process, cùng PostgreSQL transaction. Bounded context có domain/application/port/adapter và public API; không gọi HTTP giữa các module. Cross-module checkout đi qua UnitOfWork/transactional port, không ghi trực tiếp repository nội bộ.

## Consequences

Dễ giữ consistency và test race, ít hạ tầng. Cần architecture test để ngăn monolith thành code chung không ranh giới. Worker và API có thể scale process riêng; tách service chỉ sau ADR về ownership, event, SLO và distributed consistency.

## Alternatives

Microservices từ đầu tăng độ phức tạp, transaction phân tán và vận hành trước khi có bằng chứng. Một app full-stack không tách API/worker làm khó bảo vệ module và job dài.
