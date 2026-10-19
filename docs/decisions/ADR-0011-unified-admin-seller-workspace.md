---
doc_id: ADR-0011
title: Unified Admin and Seller Workspace
status: accepted
owner: Product & Architecture
last_updated: 2026-10-19
version: 1.0.0
source_of_truth: Một ứng dụng quản trị cho Seller và Platform Admin
---

# ADR-0011: Một ứng dụng quản trị, phân quyền theo vai trò và tài nguyên

## Context

F1 cần Buyer Web và giao diện quản trị cho Seller lẫn nhân sự Cartasy. Một ứng dụng quản trị chung giúp hai nhóm dùng cùng identity/session, UI primitives và quy trình build/deploy. Sự khác biệt quyền được xử lý bằng RBAC, shop capability và kiểm tra tài nguyên.

## Decision

F1 có **bốn process**: `apps/web`, `apps/admin`, `apps/api`, `apps/worker`. `apps/admin` là một Next.js app/origin/source tree với hai khu vực route: `/seller/*` cho shop onboarding/vận hành và `/platform/*` cho quản trị nền tảng. Một identity có thể là Buyer, member nhiều shop và/hoặc Platform Admin; đăng nhập bằng cùng Better Auth session. App shell lấy actor + platform capabilities + active shop memberships từ API để dựng navigation, chuyển shop và từ chối điều hướng không hợp lệ.

RBAC cấp role/capability nền tảng; membership/capability gắn `shop_id`; ABAC kiểm shop active, resource ownership/state, actor relationship, MFA/recent auth và policy theo hành động. **API/application layer là điểm quyết định quyền** trên mọi request, query, SSE, export và mutation. Ẩn route/nút ở frontend chỉ là UX. `/v1/seller/*` và `/v1/admin/*` vẫn là hai namespace API vì chúng có policy và response khác nhau, dù cùng một frontend gọi; role Platform Admin không tự có quyền thao tác thay Seller. Khi identity có cả hai vai trò, đổi workspace/shop phải clear cache, form và stream theo actor/role/shop; tránh cross-tenant data leak.

## Consequences

Chỉ một codebase/build/origin quản trị, bớt vận hành; trong cùng app phải giữ route group, feature module, API client và test permission tách rõ. F1 gate kiểm Seller chỉ thấy shop mình, Platform Admin chỉ thấy capability được cấp, user nhiều vai trò chuyển workspace an toàn, revoke/suspend có hiệu lực request kế tiếp và deep link không vượt API policy. Các flow Seller không biến thành thao tác Admin mặc định; billing/ads tương lai cần capability riêng theo [ADR-0010](ADR-0010-simulated-commerce-real-monetization-boundary.md). [Admin guide](../apps/admin/README.md) sở hữu source tree và setup; [Security](../system/security-privacy.md) sở hữu policy.
