# Research card

File này nằm cố định ở `report/team/research_card.md`; `python -m lab8.check_submission` đọc cột "Nhóm điền" bằng máy.

Phần này nối tiếp phần mini research ở buổi lý thuyết (slide 62–63). Chọn **một** câu R1–R4, điền 6 dòng, rồi biến nó thành **một ablation** chạy được trong lab: giữ nguyên model (MockLLM), tool, seed, corpus và giá, chỉ thay đổi một biến.

| Câu | Nguồn (thư viện tham khảo, slide 68–69) | Câu hỏi gợi ý cho lab |
| --- | --- | --- |
| R1 · Scaling | Google Research 2026, *Towards a Science of Scaling Agent Systems* | Task claims chủ yếu tuần tự hay song song được? Dự đoán thêm investigator sẽ làm pass rate và cost/success thay đổi thế nào. |
| R2 · Failure taxonomy | MAST, NeurIPS 2025, *Why Do Multi-Agent LLM Systems Fail?* | Các trial fail của swarm (seed 55) thuộc nhóm nào: lỗi giao việc, lệch mục tiêu, hay lỗi kiểm chứng? |
| R3 · Architecture choices | AgentArch, AACL 2026 | Nếu đổi topology mà giữ nguyên mọi thứ khác, biến nào phải cố định để kết luận còn đúng? |
| R4 · Evaluation confounders | EMNLP 2026, *Rethinking Evaluation of MAS Efficiency* | Làm sao chứng minh chênh lệch giữa A/B/C/D là do topology, không phải do prompt, tool hay seed? |

## Phiếu của nhóm

Điền vào cột cuối. Không xóa hai cột đầu.

| Field | Hướng dẫn | Nhóm điền |
| --- | --- | --- |
| CÂU | R1 / R2 / R3 / R4 | R1 |
| SOURCE | Paper/blog, năm, URL, loại bằng chứng (benchmark, production report, blog kinh nghiệm) | Google Research, Google DeepMind & MIT (2026), *Towards a Science of Scaling Agent Systems* (research.google / arxiv.org; slide 15, 24, 33, 41, 68–69); bằng chứng benchmark thực nghiệm trên 180 cấu hình SAS/MAS |
| CLAIM | Một kết luận gốc của nguồn. Không viết rộng hơn phạm vi paper | Multi-agent chỉ mang lại lợi ích khi task có thể phân tách song song; với task lập kế hoạch tuần tự, các biến thể multi-agent giảm hiệu năng 39–70% do chi phí phối hợp và lỗi tích lũy, đồng thời có ngưỡng bão hòa năng lực khi single-agent baseline đã đạt độ chính xác cao |
| BOUNDARY | Model / dataset / setup giới hạn kết luận đó | Kết luận rút ra trên các LLM sản xuất (OpenAI, Anthropic, Google Gemini) và 4 bộ benchmark (tài chính, web navigation, game planning, SWE-bench), không áp dụng trực tiếp cho MockLLM hay hệ thống mà công thức tiền/kiểm duyệt bị khóa cứng trong code host như Lab 8 |
| HYPOTHESIS | Nếu đổi shape A→B (hoặc bật/tắt một thành phần), metric X sẽ đổi theo hướng nào, khoảng bao nhiêu | Vì quy trình claim của LogiX-VN là tuần tự (`claim -> order -> policy/history -> compute_compensation -> issue_refund`), khi đổi topology từ `workflow` hoặc `single` sang `orchestrator` (thêm handoff + investigator), pass rate sẽ không tăng (giữ ở `1.0`), trong khi `mean_tool_calls` và `cost_per_success_vnd` sẽ tăng khoảng 40%–105% |
| METHOD | Biến giữ cố định: model, tools, seed, corpus, giá, evaluator. Biến duy nhất được thay đổi: … | Giữ cố định: `MockLLM`, tool registry, `as_of=2026-11-15`, seeds `[11, 22, 33, 44, 55]`, corpus `data/public_claims.jsonl` (14 claims), giá mô phỏng (`config/pricing.yaml`), evaluator `trial_pass` và `fresh_sqlite_per_trial`. Biến duy nhất thay đổi: topology (`workflow` vs `single` vs `orchestrator`) |
| FAILURE / METRIC | Một lỗi dự kiến và cách đo nó: pass rate, pass^5, variance, cost/success, leak, bypass | Lỗi dự kiến: chi phí điều phối và gọi tool tăng thừa ở bài toán tuần tự mà không tăng độ chính xác (hoặc rò secret nếu handoff thiếu allowlist). Đo bằng: `pass_rate`, `pass_5`, `seed_variance`, `cost_per_success_vnd`, `mean_steps`, `mean_tool_calls`, và `leak_suspect` / `forbidden_substring` |

## Kết quả ablation
- Lệnh đã chạy (copy nguyên văn): `python -m lab8.benchmark --shapes workflow,single,orchestrator --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/ablation_r1.json`
- Run id / file report: `src/reports/ablation_r1.json` (70 trials/shape; ví dụ `R-workflow-C-001-s11`, `R-single-C-001-s11`, `R-orchestrator-C-001-s11`) và đối chiếu tự kiểm `src/reports/hidden_selfcheck.json` (`R-orchestrator-C-H01-s11`, 200 unlabeled trials)
- Kết quả so với hypothesis (đúng / sai / không kết luận được), kèm số liệu: **Đúng với hypothesis về chi phí phối hợp và bão hòa năng lực** trên bộ public (`src/reports/ablation_r1.json`): cả `workflow`, `single` và `orchestrator` đều đạt `pass_rate = 1.0` (70/70), `pass_5 = 1.0`, `seed_min = 1.0`, `seed_max = 1.0`, `seed_variance = 0.0` (không có chênh lệch độ chính xác). Trong khi đó, `cost_per_success_vnd` tăng từ `27.86` VND (`workflow`, `mean_tool_calls = 2.79`, `mean_steps = 4.43`) và `40.27` VND (`single`, `mean_tool_calls = 3.64`, `mean_steps = 3.64`) lên `56.65` VND ở `orchestrator` (`mean_tool_calls = 5.57`, `mean_steps = 5.36`, tổng `3965.3` VND) — tức tăng `+103.3%` so với `workflow` và `+40.7%` so với `single`. Khác với mức giảm 39–70% pass rate trong paper gốc, `orchestrator` không tụt pass rate vì phần tính tiền (`compute_compensation`) và ghi sổ (`durability.issue_refund`) bị khóa trong code thay vì giao cho LLM tự lập kế hoạch; trên `student_drop` (`src/reports/hidden_selfcheck.json`), `orchestrator` cũng đạt `0` `error_trials` và `0` `leak_suspect_trials` (pass rate hidden: `unknown`).
- Giới hạn: MockLLM không phải model sản xuất, latency là `unavailable`. Bộ ablation có nhãn chỉ gồm 14 claim public nên chưa đo được hiệu năng phân biệt trên 40 claim hidden (`hidden_grade.json` là `unknown`).
