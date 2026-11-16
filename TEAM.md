# Team

## Thông tin nhóm

- Tên nhóm:
- Lớp: L3A
- Repo nhóm: `K5-L3A-DAY08-Spiritup-FromWorkflowsToAgents` (xem [SUBMISSION.md](SUBMISSION.md))
- URL repo, nhánh nộp, commit chốt:

## Thành viên

| STT | Họ tên      | MSSV        | Email                     | GitHub  | Vai trò                                                                         |
| --: | ----------- | ----------- | ------------------------- | ------- | ------------------------------------------------------------------------------- |
|   1 | Hồ Văn Dũng | 2A202603458 | 26ai.dunghv@vinuni.edu.vn | DungZai | Phần A — workflow: triển khai luồng xử lý claim cố định và kiểm tra retry/crash |
|   2 | Nguyễn Trường Thành | 2A202603512 | 26ai.thanhnt6@vinuni.edu.vn | Thanhnt-da | Phần C — orchestrator + handoff: orchestrator, investigator chỉ đọc, handoff allowlist, test ranh giới bảo mật |

Gợi ý chia vai:

- Nhóm 6 người: integration; workflow; single agent; orchestrator; ledger/policy/crash; eval/report.
- Nhóm 4 người: workflow + integration; single; orchestrator + handoff; ledger + evaluation.

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người       | Vai                | Commit                                              | Test đã thêm hoặc sửa                                                                                                                   |
| ----------- | ------------------ | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Hồ Văn Dũng | Shape A — workflow | `bfed7ad` — triển khai workflow và test retry/crash | `src/tests/test_workflow.py`: `test_workflow_unknown_claim_returns_safe_trace`, `test_workflow_resume_all_crash_windows` (3 điểm crash) |
| Nguyễn Trường Thành | Shape C — orchestrator | `5bf3daa` — triển khai orchestrator + handoff allowlist | `src/tests/test_orchestrator_boundary.py`: 18 test (status mọi nhánh, trace/result không lộ secret/PII, handoff chỉ tool đọc) |

## Cá nhân

Mỗi thành viên đồng thời tạo một báo cáo riêng trong `report/` theo dạng:

```text
report/team/MSSV_HoTen.md
```

Ví dụ: `report/team/20241234_NguyenVanAn.md`.

Sao chép mục dưới đây cho từng thành viên.

## 2A202603458_HoVanDung

### Phần việc phụ trách

- Phần A — deterministic workflow trong `src/lab8/shapes/workflow.py`.
- Kiểm tra riêng workflow bằng ma trận public và test bổ sung trong `src/tests/test_workflow.py`.

### Đóng góp cụ thể

- Thay stub bằng quy trình cố định: đọc claim qua `get_claim_view`, kiểm tra mã đơn và quyền sở hữu qua `lookup_order`, tính quote bằng `compute_compensation`, gọi host và trả `RunResult` qua `finish`.
- Dùng dữ liệu đơn có cấu trúc để tính quyền lợi; không dùng số ngày hoặc chỉ dẫn trong ticket để quyết định tiền.
- Truyền quote đầy đủ vào `issue_refund`; host kiểm approval, trần theo đơn, phần đã hoàn và idempotency. Giữ `proposal_amount` là quote ban đầu.
- Truyền `crash_at` cho host và để `InjectedCrash` truyền ra; đóng connection trong `finally`.
- Ghi trace ở các nhánh trả về và trước khi lỗi crash truyền ra; không ghi ticket, nguyên claim/order, secret hoặc thông tin cá nhân.
- Thêm test claim không tồn tại và test crash/resume tại `before_effect`, `after_effect_before_checkpoint`, `after_checkpoint`.

### Bằng chứng

- Code: `src/lab8/shapes/workflow.py`.
- Test bổ sung: `src/tests/test_workflow.py`.
- Ma trận student riêng workflow: `62 passed, 244 deselected`.
- Cổng test mặc định sau khi thêm test phần A: `58 passed, 248 deselected`.
- Môi trường kiểm tra: Python 3.12, pytest và PyYAML theo giới hạn trong `src/requirements.txt`, chạy từ `src/` bằng uv:

```text
uv run --python 3.12 --with 'pytest>=8.3.4,<9' --with 'PyYAML>=6.0.2,<7' python -m pytest -q -m student -o addopts= -k workflow
uv run --python 3.12 --with 'pytest>=8.3.4,<9' --with 'PyYAML>=6.0.2,<7' python -m pytest -q
```

- Commit phần A: `bfed7ad` trên nhánh `features/workflow`.
- Báo cáo cá nhân: `report/team/2A202603458_HoVanDung.md`. Email: `26ai.dunghv@vinuni.edu.vn`; GitHub: `DungZai`.
