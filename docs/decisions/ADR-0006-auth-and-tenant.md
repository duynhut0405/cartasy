---
doc_id: ADR-0006
title: Authentication and Shop Authorization
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Auth/session/MFA và tenant policy
review_cycle: Khi thêm provider hoặc đổi quyền shop
---

# ADR-0006: Better Auth và shop policy

## Context

Credential/session/MFA là bề mặt nhạy cảm; seller F1 tạo ranh giới tenant không thể giải quyết bằng ẩn nút ở UI.

## Decision

Better Auth self-hosted trên NestJS Express adapter, PostgreSQL session, email verification/recovery và TOTP admin/seller. Business route deny-by-default. Role global và shop membership/capability là hai tầng khác nhau; mỗi application use case kiểm tra actor, shop status, scope và resource owner. Không auto-link OAuth account cùng email. Không để frontend hoặc auth profile ghi role/shop ID server-owned.

## Consequences

Giảm tự triển khai primitive auth nhưng phải pin/test payload provider và thứ tự middleware. Seller route cần cross-tenant integration test với ít nhất hai shop ở F1.
