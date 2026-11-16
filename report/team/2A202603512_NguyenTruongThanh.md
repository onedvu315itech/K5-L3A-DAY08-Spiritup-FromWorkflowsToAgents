# Báo cáo cá nhân — 2A202603512_NguyenTruongThanh

## 1. Thông tin thành viên

- Họ tên: Nguyễn Trường Thành
- MSSV: 2A202603512
- Email: 26ai.thanhnt6@vinuni.edu.vn
- GitHub: Thanhnt-da

## 2. Phần việc phụ trách

Phần C — shape orchestrator (orchestrator + investigator chỉ đọc, handoff qua allowlist) trong `src/lab8/shapes/orchestrator.py`, và test ranh giới bảo mật của shape này trong `src/tests/test_orchestrator_boundary.py`.

## 3. Đóng góp cụ thể

- **Commit chính:**
  - `5bf3daa` — triển khai shape orchestrator (merge vào `main` qua PR #1, merge commit `e56dc21`).
  - `e483108` — thêm test ranh giới handoff/trace/result (merge vào `main` qua PR #6, merge commit `1c92e4a`).
- **Luồng xử lý của orchestrator:**
  1. Đọc claim qua view đã lọc `get_claim_view` (không `SELECT *`, nên không đụng tới `parent_secret`). Claim không tồn tại → `failed`; thiếu `order_id` → `needs_info`, không đoán mã đơn từ ticket.
  2. Dựng handoff bằng `make_handoff`, chỉ cấp 4 tool đọc: `lookup_order`, `lookup_policy`, `lookup_refund_history`, `compute_compensation`. Không cấp tool ghi.
  3. Investigator (`run_readonly_investigators`) đọc đơn theo principal. Đơn không thấy (không tồn tại hoặc thuộc principal khác) → `escalated`.
  4. Tiền do code tính: quote lấy từ `compute_compensation` trên dòng order, bỏ qua số ngày trễ ghi trong ticket. Quote = 0 → `denied`, không gọi host.
  5. Quote > 0 → gọi `durability.issue_refund` với **quote đầy đủ** và `crash_at`. Host kiểm approval, trần theo đơn, idempotency. Host từ chối → `denied` (host tự ghi `policy_denied`); conflict → `failed`; chấp nhận → `resolved`. `proposal_amount` luôn là quote đầy đủ.
  6. Mọi nhánh trả kết quả qua `finish()` nên luôn ghi trace đúng contract v1. `InjectedCrash` được ghi trace rồi ném tiếp, không bị bắt (T12).
- **Quyết định kỹ thuật:**
  - **`facts` của handoff để trống có chủ đích.** Investigator tự đọc dòng order, nên không có gì từ ticket hay parent context đi qua ranh giới. Không dùng LLM để "tóm tắt an toàn", vì tóm tắt bằng LLM không phải redaction; chỉ allowlist trong code mới tính.
  - **Không tự trừ phần đã hoàn.** Truyền quote đầy đủ vì host đã trừ phần các claim trước nhận cho cùng đơn; tự trừ trước sẽ bị trừ hai lần (T14 fail).
  - **Trace chỉ ghi dữ liệu đã lọc:** claim có `order_id` hay không, handoff đã sanitize, các field order đã qua allowlist của tool (status, delay, phí, carrier, damage), số policy hiện hành, số tiền đã hoàn trước đó, quote, phán quyết của host và quyết định cuối. **Cố ý không ghi:** ticket, nguyên claim row (có `parent_secret`, header), nguyên order row (có SĐT, địa chỉ, ghi chú nội bộ, tên hàng).
  - MockLLM chỉ dùng để soạn câu trả lời (`purpose="draft"`), không bao giờ thấy hay chọn số tiền.
  - Lỗi bất ngờ khác được chuyển thành `failed` thay vì ném exception, vì `run_claim` không được ném lỗi.
- **Test đã thêm** (`src/tests/test_orchestrator_boundary.py`, 18 test, chạy trong cổng mặc định):
  - `test_orchestrator_status_per_path`: 6 claim phủ mọi nhánh trả về — C-001 `resolved`, C-002 `denied` (ticket nói dối số ngày), C-009 `denied` (injection "ignore policy"), C-013 `denied` do host (đơn đã hoàn), C-004 `needs_info`, C-005 `escalated`.
  - `test_orchestrator_trace_and_result_have_no_secret_or_pii`: trace và `RunResult` không chứa canary, `parent_secret`, header, ticket, hay bất kỳ SĐT/địa chỉ/ghi chú nội bộ/tên hàng nào trong world.
  - `test_orchestrator_handoff_is_readonly_and_allowlisted`: bắt handoff gửi cho investigator; chỉ có field allowlist, chỉ tool đọc, không tool ghi, `facts` trong `FACT_KEYS`, không mang secret/ticket/PII/`principal_id`.
  - `test_orchestrator_needs_info_skips_handoff`: thiếu mã đơn thì không tạo handoff, không có refund.

## 4. Bằng chứng

- Code: `src/lab8/shapes/orchestrator.py` (commit `5bf3daa`).
- Test: `src/tests/test_orchestrator_boundary.py` (commit `e483108`).
- Kết quả trên `main` tại commit `1c92e4a` (Python 3.14.7, Windows):
  - Ma trận student riêng orchestrator: `62 passed` (`python -m pytest -q -m student -o addopts= -k orchestrator`).
  - Cổng mặc định: xanh, không có dòng `failed` (`python -m pytest -q`).
  - Test riêng: `18 passed` (`python -m pytest -q tests/test_orchestrator_boundary.py`).
  - `python -m lab8.check_submission`: `shape orchestrator: format đúng, 12/12 claim public pass ở seed 11`.
- **Kiểm tra test có bắt được lỗi thật (mutation):** sửa tạm orchestrator để ghi nguyên claim view (có ticket) vào trace → 6 test fail; khôi phục lại → 18 passed.
- Số liệu orchestrator trong `src/reports/benchmark.json` (14 claim × 5 seed `11,22,33,44,55`, MockLLM, giá mô phỏng):

  | Shape | Pass rate | pass^5 | Seed variance | Cost/success (VND mô phỏng) | Mean steps | Mean model calls |
  | --- | ---: | ---: | ---: | ---: | ---: | ---: |
  | orchestrator | 1.000 | 1.000 | 0.000 | 56.65 | 5.36 | 1.0 |
  | workflow (đối chứng) | 1.000 | 1.000 | 0.000 | 27.86 | 4.43 | 0.0 |
  | swarm (đối chứng) | 0.986 | 0.929 | 0.0008 | 49.57 | 2.80 | 3.0 |

  Orchestrator pass hết như workflow nhưng cost/success cao gần gấp đôi do thêm tool call của investigator. Không lặp lại lỗi của swarm ở seed 55 (tin số ngày trễ trong ticket), vì quote luôn tính từ dòng order.
- Run id tự chạy: `R-orchestrator-C-001-s11` (xem mục 5).

## 5. Công cụ AI đã dùng

- **Claude Code (Anthropic):** giải thích đề bài, luật nghiệp vụ và khác biệt giữa các shape; viết bản đầu của `orchestrator.py` và `test_orchestrator_boundary.py`; chạy ma trận student, cổng mặc định, benchmark riêng orchestrator và quét trace tìm secret/PII; hướng dẫn quy trình git (branch, PR, merge commit, tránh bẫy PR sang repo gốc khi làm trên fork).
- **Cách mình tự kiểm tra:**
  - Đọc code và đối chiếu từng nhánh với bảng trong `docs/business_rules.md` và `docs/contracts.md`.
  - Tự chạy lại trong terminal VS Code, từ `src/`:
    ```text
    python -m pytest -q -m student -o addopts= -k orchestrator
    python -m pytest -q tests/test_orchestrator_boundary.py
    python -m lab8.run --shape orchestrator --claim-id C-001 --seed 11 --db data/orch_demo.sqlite
    ```
  - Kết quả khớp `expected` của C-001: `status` `resolved`, `proposal_amount` 35000, một phần tử `refund_effects` với `amount_vnd` 35000 (`idempotency_key` `refund:C-001`), `committed_audit_count` 1, `error_reason` null.
  - Chạy lại lần hai trên cùng DB (mô phỏng retry): vẫn một dòng ledger và một audit `refund_committed`, đúng idempotency (T11).
  - Mở trace `traces/R-orchestrator-C-001-s11.json`: các event `claim`, `handoff`, `investigator`, `quote`, `host`, `decision`; không có ticket, `CANARY_SECRET_`, `parent_secret`, SĐT, địa chỉ, ghi chú nội bộ hay tên hàng.
  - Xem kết quả mutation test (mục 4) để chắc test không "luôn xanh".

## 6. Tự đánh giá

- Hiểu thêm: checkpoint không đảm bảo chỉ trả một lần, vì crash có thể rơi đúng vào giữa lúc đã ghi refund và lúc ghi checkpoint. Host chống trùng bằng ledger với khóa refund:{claim_id} (một dòng mỗi claim) và ghi refund cùng audit trong một transaction; chạy lại thì thấy dòng cũ nên không trả lần hai.
- Khó khăn: gặp khó khăn trong quy trình dùng git/Github khi làm trên repo fork, lần đầu mở PR bị trỏ sang repo gốc, phải đóng lại và mở trên repo nhóm.
- Còn hạn chế: Còn phụ thuộc vào AI phần code cho đúng code và đúng tiến độ. Sẽ đọc lại từng phần code để hiểu quy trình.
