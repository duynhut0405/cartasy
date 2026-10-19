---
doc_id: ADR-0005
title: Transactional Outbox and Idempotent Worker
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 2.0.0
source_of_truth: Giao tiếp bất đồng bộ và nhất quán event
review_cycle: Khi outbox không đạt SLO
---

# ADR-0005: Outbox PostgreSQL

## Context

Order/payment/stock cần side effect email, notification, search và analytics, nhưng gửi trực tiếp trong transaction làm lỗi hoặc kéo dài lock. Queue riêng không thể tự bảo đảm atomicity với PostgreSQL.

## Decision

Ghi outbox event versioned trong transaction nghiệp vụ. Worker poll `FOR UPDATE SKIP LOCKED`, dispatch at-least-once, consumer dedupe bằng `event_id`/`inbox_events`, retry giới hạn và failed state có alert. Payload nhỏ, không PII/secret. Domain event vận hành tách khỏi analytics consented projection.

## Consequences

Không cần Redis/BullMQ ở F1, dễ đối soát với order. Cần theo dõi backlog, cleanup/retention và replay. Khi throughput yêu cầu queue riêng, outbox vẫn là nguồn sau commit và cần ADR chuyển delivery.
