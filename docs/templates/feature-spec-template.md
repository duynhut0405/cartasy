---
doc_id: FEATURE-UNIQUE
title: Feature Name
status: draft
owner: Cartasy Product & Engineering
last_updated: YYYY-MM-DD
version: 0.1.0
source_of_truth: Hành vi duy nhất của feature
review_cycle: Khi đổi feature
---

# Tên feature

## Vấn đề và phạm vi

Actor, giá trị, giai đoạn F1/F2/F3, điều nằm ngoài phạm vi.

## Luồng chính và nhánh lỗi

Precondition → input → quyết định server → state transition → output. Ghi retry, timeout, cancel và idempotency nếu có.

## Data/API/permission

Module owner, entity/FK/index, endpoint/request/response/error, actor/shop scope, audit/outbox/analytics.

## UI và nội dung

Route, loading/empty/error, mobile, accessibility, copy mô phỏng ở checkout/order/email nếu liên quan.

## Rủi ro và kiểm thử

Unit/integration/contract/E2E/security/performance phù hợp; acceptance có thể kiểm chứng.

## Rollout

Migration N/N-1, feature flag, metric/alert, rollback/forward fix.
