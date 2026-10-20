---
name: dev-test
description: "Viết và chạy gate kiểm thử Cartasy cho vertical slice, bug fix hoặc release: integration, contract, E2E, race và security theo rủi ro thay đổi."
---

# Dev Test

- Đọc `docs/platform/testing-quality.md`, domain spec liên quan và `docs/delivery/build-checklist.md`; chọn tầng test chứng minh đúng rủi ro, không chạy gate hình thức.
- Test transaction/constraint/idempotency trên PostgreSQL thật với migration thật; kiểm OpenAPI, DTO và generated client không drift. Chỉ fake clock/ID và provider ngoài khi phù hợp.
- Với auth, shop, giá, stock, checkout, payment, event: bao phủ tenant isolation, race/retry, lỗi và replay. E2E Web/Admin dùng API thật; kiểm hai shop, hai buyer và `/vi`/`/en` theo slice.
- Không dùng `--passWithNoTests` hoặc coverage % làm bằng chứng hoàn thành. Ghi command, kết quả, commit, migration, thời gian và artifact vào checklist trước khi đánh dấu xong.
