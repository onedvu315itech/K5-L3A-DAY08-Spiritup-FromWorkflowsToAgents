# Decision record

Điền sau dấu `:` trên cùng dòng, không xóa nhãn. Giảng viên và `python -m lab8.check_submission` (chạy trong `src/`) đọc các dòng này bằng máy. Không đổi tên hay dời file này khỏi `report/team/`.
`Shape chọn` phải là đúng một trong: `workflow`, `single`, `orchestrator`, `swarm`. Đây là shape được chấm cổng hidden.

- Shape chọn: orchestrator
- Commit hash: d726848
- Run id dẫn chứng: R-orchestrator-C-001-s11, R-orchestrator-C-002-s55, R-orchestrator-C-012-s11, R-orchestrator-C-013-s11, R-orchestrator-C-014-s11 (src/reports/benchmark.json); R-orchestrator-C-H01-s11 (src/reports/hidden_selfcheck.json)
- Pass rate / pass^5 / seed min-max / variance: 1.0 (70/70) / 1.0 / 1.0–1.0 / 0.0 (public, src/reports/benchmark.json); hidden self-check: 200/200 unlabeled, 0 errors (src/reports/hidden_selfcheck.json); hidden graded: unknown
- Cost per success (mô phỏng, VND): 56.65 (3965.3 / 70, src/reports/benchmark.json); hidden total_simulated_cost_vnd: 11836.5 across 200 trials, cost/success: unknown (src/reports/hidden_selfcheck.json)
- Crash safety: 20/20 (src/reports/crash.json)
- Secret leak / approval bypass: 0/70 public (src/reports/benchmark.json), 0/200 leak_suspect on student_drop (src/reports/hidden_selfcheck.json), CR-19 pass (src/reports/crash.json); hidden approval_bypass_count: unknown

Ví dụ một dòng đã điền: `- Shape chọn: workflow`, `- Crash safety: 20/20 (src/reports/crash.json)`.

## Vì sao shape này

So trên cùng bộ 14 claim public (`C-001`–`C-014`), cùng 5 seed (`11, 22, 33, 44, 55`), cùng MockLLM, cùng bảng giá mô phỏng và cô lập mỗi trial một SQLite mới (`src/reports/benchmark.json`), `orchestrator` đạt pass rate `1.0` (70/70), pass^5 `1.0`, seed min–max `1.0–1.0` và variance `0.0` (ngang `workflow` và `single`, cao hơn `swarm` vốn đạt `0.986`, pass^5 `0.929` và variance `0.000816` do fail ở `C-002` seed `55`). Ngoài ra, khi tự kiểm trên 40 claim hidden chưa gắn nhãn trong `student_drop/` × 5 seed (`src/reports/hidden_selfcheck.json`), `orchestrator` chạy trọn vẹn 200/200 trial với `0` `error_trials` (`denied: 95`, `resolved: 90`, `escalated: 10`, `needs_info: 5`), `mean_steps: 5.5` (`p95_steps: 6.0`) và `0` `leak_suspect_trials`. Nhóm chọn `orchestrator` vì giữ được ranh giới bảo mật rõ ràng giữa orchestrator (nắm quyền gọi `durability.issue_refund`) và investigator chỉ đọc qua `handoff.v1` đã sanitize; nhược điểm là cost per success trên bộ public cao hơn (`56.65` VND so với `27.86` VND của `workflow` và `40.27` VND của `single`) và điểm chấm hidden chính thức (`hidden_grade.json`) hiện là `unknown`.

## Có đáng đổi lấy multi-agent không

Trên bộ public (`src/reports/benchmark.json`), `orchestrator` có pass rate ngang `workflow` (`1.0`) nhưng cost per success cao hơn (`56.65` so với `27.86` VND, tổng `3965.3` so với `1950.0` VND); so với `swarm` (`0.986` pass rate, variance `0.000816`, `49.57` VND/success), `orchestrator` đắt hơn một chút nhưng loại bỏ được lỗi lệch seed và giữ được ranh giới handoff chỉ đọc. Nếu ở bài toán có pass nhỉnh hơn mà cost và variance đều cao hơn, nhóm sẽ ưu tiên giữ kiến trúc ổn định và rẻ hơn (`workflow`) trừ khi chênh lệch pass trên cả 5 seed đủ lớn để vượt chi phí phát sinh. Quyết định chọn `orchestrator` sẽ đảo lại về `workflow` nếu trên bộ hidden: (1) `workflow` cũng đạt pass rate ≥ `0.75` (hoặc ngang `orchestrator`) trong khi cost per success của `orchestrator` cao gấp đôi không mang lại lợi ích độ chính xác, hoặc (2) `orchestrator` có variance tăng/vi phạm một trong 3 cổng must-pass (`approval_bypass_count = 0`, `secret_leak_count = 0`, `crash_safety = 20/20`).

## Trace và bằng chứng

Trace của `orchestrator` (`src/lab8/shapes/orchestrator.py`) ghi theo `Trace contract v1` gồm các event: `claim` (`claim_id`, `has_order_id`), `handoff` (gói `handoff.v1` đã sanitize với `facts: {}` và `allowed_tools` chỉ gồm tool đọc), `investigator` (`order_visible`, allowlist của `order` hoặc `reason`, `policy_count`, `refunded_vnd_before`), `quote` (`amount_vnd`, `reason`, `clause_ids`, `parts`), `host` (`ok`, `reason`, `conflict`), `decision` (`status`, `proposal_amount`, `reason`), kèm `crash` (`at`) trước khi re-raise `InjectedCrash` hoặc `error` (`type`) nếu có ngoại lệ. Nhóm cố ý **không** ghi `parent_secret`, `raw_headers_json`, `ticket`, `history`, hay các field nhạy cảm của `orders` (`phone`, `address`, `internal_note`, `item_name`) vào handoff hay trace; `facts` được để `{}` để nội dung ticket của người dùng không đi qua ranh giới sang investigator. Theo cơ chế fail-closed trong `docs/contracts.md` và `src/lab8/metrics.py`, grader chỉ quét `RunResult.trace_path` và các file `<run_id>*` cùng thư mục; nếu trace nằm ngoài chỗ scanner quét được (hoặc thiếu, cũ, sai định dạng), đội security **không thể** chứng minh hệ thống không lộ secret và trial bị chấm fail (`missing_trace` / `stale_trace` / `invalid_trace`).

## Giới hạn

MockLLM không phải model sản xuất; đơn giá (`20` VND / 1k input tokens, `80` VND / 1k output tokens, `10` VND / tool call) là chi phí mô phỏng (`simulated_not_real_api_price`), không phải hóa đơn API thực. Latency để `unavailable` (không tự suy diễn từ replay). Số liệu có nhãn hiện chỉ dựa trên 14 claim public (`src/reports/benchmark.json`) và 20 điểm crash (`src/reports/crash.json`); `src/reports/hidden_selfcheck.json` chỉ mới tự kiểm cho `orchestrator` trên 40 claim hidden chưa gắn nhãn (`student_drop/`), nên pass rate, `pass^5`, `cost/success` và `approval_bypass_count` trên bộ hidden chấm điểm (`hidden_grade.json`) vẫn là `unknown`. Không sửa tay các file JSON trong `src/reports/` để đạt ngưỡng `0.75`.
