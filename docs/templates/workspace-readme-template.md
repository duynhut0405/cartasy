---
doc_id: TEMPLATE-WORKSPACE
title: Workspace README Template
status: template
owner: Engineering
last_updated: 2026-10-19
version: 4.0.0
source_of_truth: Mẫu README cho workspace khi triển khai
---

# Tên workspace

## Trách nhiệm

Module/context sở hữu, input/output và nội dung ngoài phạm vi. Link tới [Architecture](../system/architecture.md), [Engineering Standards](../system/source-architecture.md) và [Tech Stack](../platform/stack-dependencies.md); khi sao chép template, điều chỉnh đường dẫn theo vị trí README thực tế.

## Cấu trúc `src`

Liệt kê cây thư mục thực tế, public export/entrypoint và dependency direction. Không copy nguyên blueprint nếu chưa triển khai.

## Chạy và kiểm thử

Lệnh dev, build, typecheck, test/smoke riêng; tên biến môi trường cần thiết, không ghi secret.

## Cách thêm feature

Owner module, schema/port/adapter/route/UI theo thứ tự, test/gate và tài liệu cần đồng bộ. Trạng thái hoàn thành chỉ ghi khi có bằng chứng.
