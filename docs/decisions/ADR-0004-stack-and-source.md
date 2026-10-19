---
doc_id: ADR-0004
title: TypeScript Stack and Source Boundaries
status: accepted
owner: Cartasy Engineering
last_updated: 2026-10-19
version: 2.0.0
source_of_truth: Quyết định stack và source organization
review_cycle: Trước khi nâng framework major
---

# ADR-0004: TypeScript monorepo có ranh giới

## Context

Web, admin, API và worker chia sẻ contract nhưng không được chia sẻ database entity hoặc code UI/domain tùy tiện. AI Agent cần cấu trúc cố định trước khi thêm feature.

## Decision

Node.js 24 LTS, pnpm/Turborepo, Next.js App Router, NestJS ESM, PostgreSQL và Drizzle SQL migrations. `apps/*` là composition/presentation; `packages/server` chứa domain/application/port theo bounded context; `packages/database` chứa schema/repository adapter; `packages/contracts` chứa OpenAPI/generated client. ESLint/dependency-cruiser chặn import ngược/cycle/deep import. Chi tiết và thư viện ở [Tech Stack](../platform/stack-dependencies.md), cây `src` ở [Engineering Standards](../system/source-architecture.md).

## Consequences

Dễ thay adapter và phát triển web/admin song song nhưng cần giữ public export và contract drift gate. Không tạo package/module chung chỉ vì một feature mới cần đặt file.
