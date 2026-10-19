---
doc_id: ADR-0007
title: Portable Node Container Deployment
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 2.0.0
source_of_truth: Hosting baseline và tách môi trường
review_cycle: Khi chọn provider hoặc đổi runtime
---

# ADR-0007: Container Node portable

## Context

Dự án cần Next.js/NestJS/worker, PostgreSQL, object storage, SMTP và staging/production riêng. Quyết định vendor quá sớm sẽ buộc code vào adapter đặc thù trước khi có domain/traffic thật.

## Decision

Build/deploy web, admin, API, worker như Node.js container; PostgreSQL managed, S3-compatible private bucket và SMTP adapter. CI chạy gate, migration job một lần, production có approval và restore drill. Local dùng Compose. Chọn nhà cung cấp bằng khả năng đáp ứng SLO, backup/region/chi phí và smoke trên HTTPS; không đặt provider name vào domain code.

## Consequences

Có thể chuyển hosting với ít đổi code nhưng phải tự xác minh networking, cookie, cache, secret và vận hành của provider được chọn. Edge runtime chỉ qua ADR riêng sau khi chứng minh tương thích với route/auth.
