# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Số liệu lấy từ file trong `src/reports/`, không chép tay từ terminal.

## 1. Thông tin nhóm

- Tên nhóm: Spiritup
- Thành viên: xem `TEAM.md`
- Commit chốt (trùng `report/team/decision_adr.md`):

## 2. Mục tiêu và cách tiếp cận

Bài toán là xử lý claim hoàn tiền của LogiX-VN theo cùng luật nghiệp vụ trên bốn kiến trúc, sau đó chọn kiến trúc ít tự chủ nhất vẫn đáp ứng yêu cầu bằng số đo. Tiền, approval và chống ghi trùng do code kiểm soát; ticket là dữ liệu người dùng và không thể thay đổi policy.

Hồ Văn Dũng (2A202603458) phụ trách phần A — workflow trong `src/lab8/shapes/workflow.py`. Phần A dùng các bước Python cố định, làm baseline để so sánh với single agent, orchestrator và swarm. Quy trình được triển khai theo thứ tự: kiểm tra claim và quyền sở hữu, tính quote, gọi host ghi refund, ghi trace và kiểm tra retry/crash.

Vũ Lê Đức (2A202603683) phụ trách phần B — single agent trong `src/lab8/shapes/single.py` và chạy các bộ kiểm thử, benchmark (`src/reports/benchmark.json`, `src/reports/crash.json`) để tổng hợp số liệu đối chiếu.

Nguyễn Trường Thành (2A202603512) phụ trách phần C — orchestrator trong `src/lab8/shapes/orchestrator.py`: orchestrator giữ quyền quyết định và gọi host, investigator chỉ có tool đọc và nhận việc qua handoff `handoff.v1` đã lọc bằng allowlist; kèm test ranh giới bảo mật `src/tests/test_orchestrator_boundary.py`. Đây là shape nhóm chọn để chấm hidden (xem `report/team/decision_adr.md`).

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

Giữ nguyên chữ ký `run_claim`. Orchestrator lập kế hoạch và giữ quyền ghi; việc đọc dữ liệu được giao cho investigator chỉ đọc qua một handoff có kiểu:

1. **Đọc claim qua view đã lọc.** Orchestrator dùng `get_claim_view` (không `SELECT *`), nên `parent_secret` và header không bao giờ vào bộ nhớ của shape. Claim không tồn tại trả `failed`; thiếu `order_id` trả `needs_info`, không tạo handoff và không suy mã đơn từ ticket.
2. **Dựng handoff bằng allowlist trong code.** `make_handoff` chỉ giữ các field `handoff.v1` được phép và chỉ cấp 4 tool đọc: `lookup_order`, `lookup_policy`, `lookup_refund_history`, `compute_compensation`. `facts` để trống có chủ đích: investigator tự đọc dòng order, nên không có câu chữ ticket hay parent context nào đi qua ranh giới. `make_handoff` từ chối nếu còn canary hoặc tool ghi.
3. **Investigator chỉ đọc.** `run_readonly_investigators` chạy các tool trong handoff với `worker=True`; dispatcher của host chặn mọi tool ghi. Đơn không thấy (không tồn tại hoặc thuộc principal khác) trả `escalated`.
4. **Tiền do code tính.** Quote lấy từ `compute_compensation` trên dòng order và policy hiện hành; số ngày ghi trong ticket bị bỏ qua (khác swarm ở seed 55). Quote bằng 0 trả `denied`, không gọi host.
5. **Chỉ orchestrator gọi host.** Quote dương được truyền **đầy đủ** vào `durability.issue_refund(conn, claim_id, quote, crash_at=crash_at)`. Host kiểm approval đúng claim, trần theo đơn, phần đã hoàn và idempotency. Host chấp nhận trả `resolved`; host từ chối trả `denied` (host tự ghi `policy_denied`); xung đột số tiền trả `failed`. `proposal_amount` luôn là quote đầy đủ.

**Ranh giới code/model và độ bền.** MockLLM chỉ được gọi để soạn câu trả lời (`purpose="draft"`), không thấy và không chọn số tiền; mọi quyết định trạng thái và số tiền nằm trong code. `InjectedCrash` được ghi vào trace rồi ném tiếp, không bị bắt, để T12 chứng minh crash thật sự xảy ra; resume an toàn nhờ ledger một dòng mỗi claim với khóa `refund:{claim_id}`, không nhờ checkpoint. Lỗi bất ngờ khác được chuyển thành `failed` vì `run_claim` không được ném exception.

**Handoff, trace và bảo mật.** Các event trace: `claim` (`claim_id`, `has_order_id`), `handoff` (gói đã sanitize), `investigator` (`order_visible`, các field order đã qua allowlist của tool, `policy_count`, `refunded_vnd_before`), `quote`, `host` (`ok`, `reason`, `conflict`), `decision`, kèm `crash` hoặc `error` khi có. Trace ghi handoff đã sanitize, không ghi parent context. Cố ý **không** ghi ticket, nguyên claim row (`parent_secret`, header) và nguyên order row (SĐT, địa chỉ, ghi chú nội bộ, tên hàng). Tóm tắt bằng LLM không được dùng thay cho redaction; chỉ allowlist trong code được tính là lọc.

**Kiểm thử riêng.** `src/tests/test_orchestrator_boundary.py` (18 test, trong cổng mặc định) chạy 6 claim phủ mọi nhánh trả về (C-001 `resolved`; C-002, C-009, C-013 `denied`; C-004 `needs_info`; C-005 `escalated`) và kiểm: đúng status; trace và `RunResult` không chứa canary, header, ticket hay bất kỳ SĐT/địa chỉ/ghi chú/tên hàng nào; handoff chỉ có field allowlist, chỉ tool đọc, không mang `principal_id`; thiếu mã đơn thì không tạo handoff. Mutation check: sửa tạm shape để ghi nguyên claim view vào trace thì 6 test fail.

## 4. Kết quả

### Benchmark 4 shape trên bộ public

Cùng 14 claim public (`C-001`–`C-014`), cùng 5 seed `11, 22, 33, 44, 55`, cùng MockLLM, cùng bảng giá mô phỏng, mỗi trial một SQLite mới. Số liệu lấy nguyên từ `src/reports/benchmark.json`:

| Shape | Pass rate | pass^5 | Seed min–max | Variance | Cost/success (VND mô phỏng) | Mean / p95 steps | Mean model calls |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| A `workflow` | 1.000 (70/70) | 1.000 | 1.00–1.00 | 0.000000 | 27.86 | 4.43 / 5 | 0.00 |
| B `single` | 1.000 (70/70) | 1.000 | 1.00–1.00 | 0.000000 | 40.27 | 3.64 / 4 | 4.64 |
| C `orchestrator` (shape chọn) | 1.000 (70/70) | 1.000 | 1.00–1.00 | 0.000000 | 56.65 | 5.36 / 6 | 1.00 |
| D `swarm` (tham chiếu) | 0.986 (69/70) | 0.929 | 0.93–1.00 | 0.000816 | 49.57 | 2.80 / 3 | 3.00 |

Ba shape nhóm viết đều đạt pass rate `1.0` trên cả 5 seed. `swarm` fail đúng một trial (`C-002` seed `55`) do điểm yếu tham chiếu tin số ngày trễ ghi trong ticket thay vì `delay_days` của đơn; cả A, B, C đều tính tiền từ dòng order nên không gặp lỗi này. Về chi phí, `workflow` rẻ nhất (không gọi model); `orchestrator` đắt nhất do thêm tool call của investigator (`mean_tool_calls = 5.57`). Chi phí là mô phỏng, latency `unavailable`.

### Crash safety và tự kiểm hidden

- **Crash suite** (`src/reports/crash.json`): `crash_safety = 20/20`. Mỗi điểm crash, sau resume vẫn đúng một refund và một audit `refund_committed`.
- **Tự kiểm trên `student_drop/`** (`src/reports/hidden_selfcheck.json`, chỉ `orchestrator`, 40 claim × 5 seed): `200/200` trial chạy hết, `0` `error_trials`, `0` `leak_suspect_trials`; phân bố status `resolved 90`, `denied 95`, `escalated 10`, `needs_info 5`. Bộ này không có nhãn nên **chưa có pass rate hidden**; điểm chính thức do grader của giảng viên chấm.

### Kiểm thử theo shape

- **A `workflow`:** `62 passed` trên ma trận student; test bổ sung `src/tests/test_workflow.py` kiểm claim không tồn tại và crash/resume tại `before_effect`, `after_effect_before_checkpoint`, `after_checkpoint` (một refund, một audit committed sau resume). Chi tiết: [báo cáo cá nhân Hồ Văn Dũng](2A202603458_HoVanDung.md), mục 4.
- **B `single`:** `62 passed` trên ma trận student (`-k "single]"`). Chi tiết: [báo cáo cá nhân Vũ Lê Đức](2A202603683_VuLeDuc.md).
- **C `orchestrator`:** `62 passed` trên ma trận student (`-k orchestrator`) và `18 passed` ở `src/tests/test_orchestrator_boundary.py` (status mọi nhánh, trace/result không lộ secret/PII, handoff chỉ tool đọc). Chi tiết: [báo cáo cá nhân Nguyễn Trường Thành](2A202603512_NguyenTruongThanh.md).
- **Toàn bộ trên commit chốt:** cổng mặc định `python -m pytest -q` xanh (`76 passed`), ma trận student 4 shape `248 passed`, `python -m lab8.check_submission` `0 FAIL`.

### Lệnh tái lập (chạy trong `src/`)

```text
python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json
python -m lab8.crash_test --count 20 --out reports/crash.json
python -m lab8.benchmark --shapes orchestrator --cases student_drop/hidden_claims.jsonl --world student_drop/hidden_world_state.json --out reports/hidden_selfcheck.json
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

Hồ Văn Dũng — MSSV 2A202603458: triển khai workflow, ghi trace có chọn lọc, tích hợp host và thêm test claim không tồn tại cùng ba điểm crash/resume. Commit kỹ thuật: `bfed7ad`, nhánh `features/workflow`. Codex hỗ trợ triển khai và chạy test; thành viên đã tự chạy C-001 và ghi cách đối chiếu kết quả trong báo cáo cá nhân.

Vũ Lê Đức — MSSV 2A202603683: triển khai Shape B (`single`) trong `src/lab8/shapes/single.py` nhánh `features/single-react`, chạy các bộ test public và tổng hợp `crash.json`, `benchmark.json`, `ablation_r1.json`, `hidden_selfcheck.json`

Nguyễn Trường Thành — MSSV 2A202603512: triển khai Shape C (`orchestrator`) với handoff allowlist và investigator chỉ đọc, ghi trace đã lọc, tích hợp host và crash; thêm `src/tests/test_orchestrator_boundary.py` (18 test ranh giới handoff/trace/result). Commit kỹ thuật: `5bf3daa` (shape), `e483108` (test). Claude Code hỗ trợ giải thích đề, viết bản đầu và chạy kiểm tra; thành viên đã tự chạy lại test, chạy C-001 hai lần trên cùng DB (một refund, đúng idempotency) và kiểm trace, ghi trong [báo cáo cá nhân](2A202603512_NguyenTruongThanh.md).

## 7. Hạn chế và hướng cải thiện

MockLLM không phải model sản xuất; latency là `unavailable` và giá token là mô phỏng. Workflow hiện dùng dữ liệu có cấu trúc, không gọi model; vì vậy chưa đánh giá khả năng xử lý trường hợp cần hiểu nội dung tự do hoặc thu thập thêm thông tin qua hội thoại.

Kiểm tra public của phần A chưa thay thế bộ hidden và chưa đủ để kết luận workflow tốt hơn các shape khác. Khi có benchmark đầy đủ, nhóm cần cân nhắc độ đúng, độ ổn định và cost/success trên cùng corpus, seed và evaluator. Nếu task mở rộng cần xử lý thông tin mơ hồ hoặc các bước điều tra linh hoạt, lợi ích của agent cần được đo lại trước khi đổi kiến trúc.

Hạn chế của single: Dù cùng đạt độ chính xác tối đa trên bộ public, `single` gộp chung ngữ cảnh đọc ticket và luồng xử lý hoàn tiền trong một agent nên không có ranh giới cách ly bảo mật rõ ràng. Hướng cải thiện là tách riêng ngữ cảnh dữ liệu đầu vào khỏi tác nhân ra quyết định và bổ sung kiểm thử trên tập dữ liệu lớn hơn để đánh giá độ ổn định lâu dài.

Hạn chế của orchestrator: investigator trong lab là code tất định chứ không phải model, nên ranh giới handoff mới được kiểm với một worker không "sáng tạo"; với worker là LLM thật cần kiểm thêm đầu ra của worker trước khi orchestrator dùng. Cost/success cao nhất trong ba shape nhóm viết (`56.65` VND) mà không tăng pass rate trên bộ public; lợi thế của orchestrator chủ yếu là ranh giới bảo mật rõ ràng, chưa được chứng minh bằng độ đúng. Investigator chạy tuần tự; chạy song song (bonus B1) chưa làm.

## 8. Tài liệu tham khảo

- [Luật nghiệp vụ LogiX-VN](../../docs/business_rules.md).
- [Output contract và Trace contract v1](../../docs/contracts.md).
- [Kiến trúc và ranh giới code/model](../../docs/architecture.md).
- [Định nghĩa metric](../../docs/metrics.md).
- [Rubric](../../RUBRIC.md) và [quy định nộp bài](../../SUBMISSION.md).
- Mã nguồn phần A: `src/lab8/shapes/workflow.py`; test: `src/tests/test_public_matrix.py`, `src/tests/test_workflow.py`.
- Mã nguồn phần C: `src/lab8/shapes/orchestrator.py`; test: `src/tests/test_public_matrix.py`, `src/tests/test_orchestrator_boundary.py`.
