# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Số liệu lấy từ file trong `src/reports/`, không chép tay từ terminal.

## 1. Thông tin nhóm

- Tên nhóm: Spiritup
- Thành viên: xem `TEAM.md`
- Commit chốt (trùng `report/team/decision_adr.md`):

## 2. Mục tiêu và cách tiếp cận

Bài toán là xử lý claim hoàn tiền của LogiX-VN theo cùng luật nghiệp vụ trên bốn kiến trúc, sau đó chọn kiến trúc ít tự chủ nhất vẫn đáp ứng yêu cầu bằng số đo. Tiền, approval và chống ghi trùng do code kiểm soát; ticket là dữ liệu người dùng và không thể thay đổi policy.

Hồ Văn Dũng (2A202603458) phụ trách phần A — workflow trong `src/lab8/shapes/workflow.py`. Phần A dùng các bước Python cố định, làm baseline để so sánh với single agent, orchestrator và swarm. Quy trình được triển khai theo thứ tự: kiểm tra claim và quyền sở hữu, tính quote, gọi host ghi refund, ghi trace và kiểm tra retry/crash. Phân công và tiến độ các shape còn lại chờ thành viên phụ trách bổ sung.

Vũ Lê Đức (2A202603683) phụ trách phần B — single agent trong `src/lab8/shapes/single.py` và chạy các bộ kiểm thử, benchmark (`src/reports/benchmark.json`, `src/reports/crash.json`) để tổng hợp số liệu đối chiếu.

## 3. Giải pháp

### A — Workflow

Giữ nguyên chữ ký `run_claim(claim_id, *, seed, store_path, crash_at=None) -> RunResult`. Workflow mở SQLite và thực hiện:

1. Đọc claim bằng `get_claim_view`. Claim không tồn tại trả `failed`; thiếu `order_id` trả `needs_info`, không ghi refund và không suy mã đơn từ ticket.
2. Gọi `lookup_order` với `principal_id` của claim. Đơn không tồn tại hoặc ngoài quyền sở hữu trả `escalated`.
3. Gọi `compute_compensation` trên dữ liệu đơn có cấu trúc. Hàm dùng policy hiện hành, có hiệu lực tại mốc của lab. Trễ đủ điều kiện được hoàn phí ship; hỏng hàng đã xác minh được tính theo giới hạn policy. Nếu cùng đủ hai điều kiện thì lấy khoản lớn hơn. Số ngày trong ticket không thay thế `delay_days` của đơn.
4. Quote bằng 0 trả `denied` và không gọi ghi tiền. Quote dương được truyền đầy đủ vào `durability.issue_refund`. Host kiểm approval đúng claim, hiệu lực approval, trần tiền và lịch sử refund trong cơ chế ghi của host.
5. Host chấp nhận trả `resolved`; host từ chối trả `denied`; xung đột số tiền trả `failed`. Mọi nhánh bình thường dùng `finish` để lấy refund và audit thật từ ledger, ghi trace và tạo `RunResult`.

`proposal_amount` luôn là quote đầy đủ. Ví dụ T14 có entitlement 150.000 VND và đã hoàn 30.000 VND: workflow gửi 150.000 VND, host chỉ ghi thêm 120.000 VND. Workflow không tự trừ tiền cũ để tránh host trừ lần thứ hai. Với quote dương bị host chặn, gọi host vẫn cần thiết để tạo audit `policy_denied`.

**Ranh giới code/model và độ bền.** Workflow không gọi MockLLM; các bước và quyết định dựa trên code. Chỉ host ghi ledger, với idempotency key `refund:{claim_id}`. Workflow chuyển `crash_at` cho host và để `InjectedCrash` truyền ra; connection luôn đóng trong `finally`. Khi resume, host dùng refund đã commit để chống ghi trùng.

**Trace và bảo mật.** Các event là `claim_lookup`, `order_lookup`, `quote`, `refund_requested`, `decision`. Trace chỉ ghi cờ tồn tại/quyền truy cập, lý do, quote và quyết định; không ghi ticket, nguyên claim/order, secret, header, số điện thoại, địa chỉ, tên hàng hoặc ghi chú nội bộ. Trace được ghi ở các nhánh trả về và trong `finally` quanh lời gọi host nếu có crash. Shape A không chuyển việc sang agent khác nên không tạo handoff; phần handoff allowlist của orchestrator do thành viên phần C bổ sung.

### B — Single agent

Shape B sử dụng một ReAct agent qua `lab8.runtime.run_react` với `max_steps=8` (`worker=True`).

- **Vòng lặp ReAct và ranh giới tool:** Hàm `decide(obs)` đếm lời gọi mô hình qua `mock_llm.complete` vào `Budget` và chỉ gọi các tool đọc theo nhu cầu (`lookup_order`, `lookup_refund_history`, `compute_compensation`), không gọi `issue_refund` trong vòng lặp. Kết quả tính tiền dựa trên dữ liệu đơn hàng có cấu trúc thay vì tin nội dung `ticket`.
- **Giao dịch host và độ bền:** Sau vòng lặp, code xử lý các nhánh `step_cap`, `needs_info`, `escalated`, `denied` (`proposal <= 0`), và truyền quote đầy đủ vào `durability.issue_refund(..., crash_at=crash_at)` để host tự áp approval, trần theo đơn và idempotency. Khi gặp `InjectedCrash`, shape ghi trace rồi để exception truyền ra ngoài và đóng connection trong `finally`.
- **Trace an toàn:** Sử dụng `lab8.trace.Trace` và `safe_dump` để ghi các sự kiện chính (`tool`, `quote`, `decision`, `host_outcome`, `crash`, `error`), không ghi `ticket`, `parent_secret`, `raw_headers` hay thông tin cá nhân của đơn hàng.

### C — Orchestrator

Chờ thành viên phụ trách bổ sung thiết kế, ranh giới code/model và bằng chứng tương ứng.

## 4. Kết quả

Bảng so sánh 4 shape trên bộ public (từ `src/reports/benchmark.json`): pass rate, pass^5, seed min–max, variance, cost/success, mean/p95 steps. Kèm crash safety (`src/reports/crash.json`) và, nếu có, kết quả tự kiểm trên `student_drop/`.

Phần A đã được kiểm tra bằng ma trận public trong `src/tests/test_public_matrix.py` và test bổ sung trong `src/tests/test_workflow.py`. Bằng chứng kiểm thử và lệnh đã chạy được ghi trong [báo cáo cá nhân Hồ Văn Dũng](2A202603458_HoVanDung.md), mục 4. Các kiểm tra bổ sung gồm claim không tồn tại và crash/resume tại `before_effect`, `after_effect_before_checkpoint`, `after_checkpoint`, xác nhận một refund và một audit committed sau resume.

Đây là kiểm chứng triển khai, chưa phải bảng metric benchmark. Tại thời điểm bổ sung phần A, `src/reports/` chưa có `benchmark.json` và `crash.json`. Chưa kết luận pass rate hidden hoặc crash safety `20/20` từ các test riêng workflow.

Phần B (`single`) đã đạt `62 passed, 186 deselected` trên ma trận student (`tests/test_public_matrix.py -k "single]"`), cùng bộ test mặc định `58 passed` và `lab8.check_submission` đạt `0 FAIL`. Theo số liệu tổng hợp trong `src/reports/benchmark.json` (14 claim public × 5 seed = 70 trial) và `src/reports/crash.json`: shape `single` đạt `pass_rate = 1.0` (`70/70`), `pass^5 = 1.0`, `seed min–max = 1.0–1.0`, `variance = 0.0`, `cost_per_success_vnd = 40.27` VND (chi phí mô phỏng), `mean_steps = 3.64`, `p95_steps = 4.0`, và `crash_safety = 20/20`.

Nhóm cần chạy các lệnh sau trong `src/` rồi bổ sung bảng pass rate, pass^5, seed min–max, variance, cost/success, mean/p95 steps cho đủ bốn shape từ report được sinh ra:

```text
python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json
python -m lab8.crash_test --count 20 --out reports/crash.json
```

Quyết định kiến trúc và lý do: xem `report/team/decision_adr.md`. Phiếu nghiên cứu và ablation: xem `report/team/research_card.md`.

## 5. Khó khăn và cách xử lý

Khi kiểm tra phần A, terminal chưa nhận lệnh `python` trong PATH. Python 3.12 đã cài nhưng chạy pytest trực tiếp báo `No module named pytest`. Cách xử lý là dùng uv chuẩn bị pytest và PyYAML theo giới hạn trong `requirements.txt`, rồi chạy từ `src/`:

```text
uv run --python 3.12 --with 'pytest>=8.3.4,<9' --with 'PyYAML>=6.0.2,<7' python -m pytest -q -m student -o addopts= -k workflow
uv run --python 3.12 --with 'pytest>=8.3.4,<9' --with 'PyYAML>=6.0.2,<7' python -m pytest -q
```

Khó khăn nghiệp vụ là phân biệt quote với số thực hoàn và xử lý crash sau khi tiền đã ghi. Phần A giải quyết bằng cách truyền quote đầy đủ và sử dụng cơ chế transaction/idempotency sẵn có của host; không sửa host hoặc thước đo.

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`.

Hồ Văn Dũng — MSSV 2A202603458: triển khai workflow, ghi trace có chọn lọc, tích hợp host và thêm test claim không tồn tại cùng ba điểm crash/resume. Commit kỹ thuật: `bfed7ad`, nhánh `features/workflow`. Codex hỗ trợ triển khai và chạy test; thành viên đã tự chạy C-001 và ghi cách đối chiếu kết quả trong báo cáo cá nhân. Đóng góp các thành viên khác chờ nhóm bổ sung.

Vũ Lê Đức — MSSV 2A202603683: triển khai Shape B (`single`) trong `src/lab8/shapes/single.py` nhánh `features/single-react`, chạy các bộ test public và tổng hợp `crash.json`, `benchmark.json`, `ablation_r1.json`, `hidden_selfcheck.json`

## 7. Hạn chế và hướng cải thiện

MockLLM không phải model sản xuất; latency là `unavailable` và giá token là mô phỏng. Workflow hiện dùng dữ liệu có cấu trúc, không gọi model; vì vậy chưa đánh giá khả năng xử lý trường hợp cần hiểu nội dung tự do hoặc thu thập thêm thông tin qua hội thoại.

Kiểm tra public của phần A chưa thay thế bộ hidden và chưa đủ để kết luận workflow tốt hơn các shape khác. Khi có benchmark đầy đủ, nhóm cần cân nhắc độ đúng, độ ổn định và cost/success trên cùng corpus, seed và evaluator. Nếu task mở rộng cần xử lý thông tin mơ hồ hoặc các bước điều tra linh hoạt, lợi ích của agent cần được đo lại trước khi đổi kiến trúc.

Hạn chế của single: Dù cùng đạt độ chính xác tối đa trên bộ public, `single` gộp chung ngữ cảnh đọc ticket và luồng xử lý hoàn tiền trong một agent nên không có ranh giới cách ly bảo mật rõ ràng. Hướng cải thiện là tách riêng ngữ cảnh dữ liệu đầu vào khỏi tác nhân ra quyết định và bổ sung kiểm thử trên tập dữ liệu lớn hơn để đánh giá độ ổn định lâu dài.

## 8. Tài liệu tham khảo

- [Luật nghiệp vụ LogiX-VN](../../docs/business_rules.md).
- [Output contract và Trace contract v1](../../docs/contracts.md).
- [Kiến trúc và ranh giới code/model](../../docs/architecture.md).
- [Định nghĩa metric](../../docs/metrics.md).
- [Rubric](../../RUBRIC.md) và [quy định nộp bài](../../SUBMISSION.md).
- Mã nguồn phần A: `src/lab8/shapes/workflow.py`; test: `src/tests/test_public_matrix.py`, `src/tests/test_workflow.py`.
