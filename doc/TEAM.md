# Nhóm thực hiện K4 Track 3B

Tên nhóm: FAMI

Repo chung: https://github.com/lechihung252/K4-Track-3B-Day24-Eval-Guard

Trưởng nhóm/người nộp: Lê Chí Hùng — 2A202602863

Ngày lab/deadline áp dụng: 12/10/2026 23:59 — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: 9e264c5

Nhóm 3 người, mỗi người phụ trách một phase; blueprint (Task 13) do cả nhóm điền
từ kết quả A/B/C.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Lê Chí Hùng | 2A202602863 | Phase C: Tasks 9a–12 trong `src/phase_c_guard.py`, prompt input rail trong `guardrails/config.yml`, chạy batch C; tích hợp nhánh và nộp bài | Commit 6dfe4f5 (nhánh `phase-c`); `artifacts/guard_results.json` |
| 2 | Trần Tuấn Hoàng | 2A202602832 | Phase A: module m1–m5, Tasks 1–4 trong `src/phase_a_ragas.py`, tinh chỉnh m4_eval, chạy batch Phase A live | PR #1 (commits 32b492e, 6551182), PR #2 (`artifacts/answers_50q.json`); PR #5 (`artifacts/ragas_50q.json`, `doc/reports/failure_clusters.md`) |
| 3 | Nguyễn Văn Hưởng | 2A202602743 | Phụ trách Phase B: Tasks 5–8 với hỗ trợ Codex, tests offline, batch judge thật và báo cáo κ/bias | PR #3; `src/phase_b_judge.py`, `tests/test_phase_b.py`, `artifacts/judge_results.json`, `doc/reports/bias_report.md` |

Người tích hợp/chạy batch/nộp link: Lê Chí Hùng (trưởng nhóm, điều phối PR và nộp repo), Trần Tuấn Hoàng (sinh 50 answers và chạy batch Phase A live), Nguyễn Văn Hưởng (chạy batch Phase B live).

## Môi trường và chuẩn bị trước lớp

### Máy 1 — Lê Chí Hùng (Phase C)

OS/Python/compiler: macOS 26.6.2 (x86_64), Python 3.11.16, Apple clang 17.0.0
(Command Line Tools). `annoy==1.17.3` build thành công từ source.

Virtualenv/dependencies/imports: `python3.11 -m venv .venv`, rồi
`pip install -r requirements/requirements.txt -c requirements/constraints-core.txt`.
Lần đầu `pip check` báo không lỗi nhưng `import ragas` lỗi vì pip lấy pyarrow 26
(cần NumPy 2, lab pin NumPy 1.26.4); torch trên macOS x86_64 chỉ có tới 2.2.2.
Cài lại với constraints bổ sung `pyarrow<18`, `pandas<3`, `transformers<4.50`,
`huggingface-hub<1`, `sentence-transformers<4`. Sau đó `pip check` không lỗi và
`import ragas, nemoguardrails, presidio_analyzer, spacy` chạy được.

Models/Qdrant/50 answers đã chuẩn bị: spaCy `en_core_web_lg` 3.8.0 tải và load
được; Qdrant `qdrant/qdrant:v1.9.7` chạy qua Docker Compose. Full NeMo 0.10.0 chạy
thật trong batch C với `gpt-4o-mini`. 50 answers do thành viên Phase A sinh
(PR #2): `validate_answers` đạt, 20/20/10, cả 50 câu `generation_mode=llm` với
### Máy 3 — Trần Tuấn Hoàng (Phase A)

OS/Python/compiler: Windows 11 + WSL2 Ubuntu 24.04 (Linux 5.15.167.4-microsoft-standard-WSL2 x86_64), Python 3.11.11, GCC 13.3.0.
Virtualenv/dependencies/imports: Môi trường conda `(day24)` kế thừa Python 3.11. Cài đặt các gói theo `requirements/requirements.txt` và `constraints-core.txt`; hạ phiên bản `pyarrow==17.0.0` để tương thích hoàn toàn `numpy==1.26.4`. Kiểm tra `python -c "import ragas, nemoguardrails, presidio_analyzer, spacy; print('imports OK')"` cho kết quả OK. Đã tải mô hình spaCy `en_core_web_lg`.
Models/Qdrant/50 answers đã chuẩn bị: Qdrant `qdrant/qdrant:v1.9.7` chạy ổn định qua Docker Compose (cổng 6333). Sinh thành công 50 answers bằng `python scripts/setup_answers.py --live` sử dụng `gpt-4o-mini-2024-07-18` (100% LLM, không fallback). Chạy live evaluation Phase A: `python scripts/run_experiments.py --phase a --live` tạo `artifacts/ragas_50q.json` với 50 câu đánh giá đầy đủ qua Ragas 0.1.22.

### Máy 2 — Nguyễn Văn Hưởng (Phase B)

OS/Python/compiler đã kiểm chứng trên máy chuẩn bị: macOS 26.5, Apple Silicon
arm64; Python 3.11.16; Apple clang 17.0.0; Annoy 1.17.3 build thành công.
Virtualenv/dependencies/imports: `.venv` cho full runtime theo requirements và
constraints; `.venv-tests` theo lockfile offline. `pip check` đạt trên cả hai.
RAGAS/NeMo/Presidio/spaCy/sentence-transformers và các dependency còn lại import
thành công. PyArrow pin 19.0.1 để tương thích NumPy 1.26.4.
Models/Qdrant: spaCy `en_core_web_lg` 3.8.0 nạp được; Presidio khởi tạo và phát
hiện email mẫu; BGE-M3 nạp từ cache và sinh embedding (1,1024); Qdrant v1.9.7
đã kiểm tra health/API và kết nối client. Cấu hình NeMo/Colang parse thành công,
chưa kiểm chứng LLM guard qua API. Đã tích hợp 50 answers từ `origin/main`
commit `96f6b2c`: schema/ids/nguồn và phân bố 20/20/10 đạt; không chạy lại
indexing hoặc generation trên máy này. Phase B đã chạy một batch thật bằng
gpt-4o-mini; 10 binary labels khớp human, κ=1.0, position inconsistency=0/5,
verbosity win rate=5/5. Bằng chứng: `artifacts/judge_results.json`.

## INDIVIDUAL

### LeChiHung-2A202602863

- Phần được giao và phần thực tế đã làm: Phase C trong `src/phase_c_guard.py`.
  Task 9a `pii_scan` (Presidio, chỉ giữ `VN_CCCD`, `VN_PHONE`, `EMAIL_ADDRESS`);
  Task 9b `check_input_rail` và Task 11 `check_output_rail` (quyết định đọc từ
  `activated_rails[].stop`, không dò từ khóa từ chối); Task 10
  `run_adversarial_suite` (Presidio rồi NeMo input trong một event loop); Task 12
  `measure_p95_latency` (2 warm-up thật, 20 mẫu, đo đủ hai tầng). Viết lại prompt
  `self_check_input` trong `guardrails/config.yml` để xử lý chặn nhầm.
- Bằng chứng: commit 6dfe4f5 trên nhánh `phase-c`; `artifacts/guard_results.json`
  (created_at 2026-10-12T10:06:35Z, guard model `gpt-4o-mini`).
- Cách kiểm tra: `pytest tests/test_phase_c.py` 12/12 pass (mock). Chạy Presidio
  thật offline trên 28 inputs trước khi viết `pii_scan`. Batch live
  `run_experiments.py --phase c --live --latency-samples 20` hai lần: lần 1 attacks
  18/20, clean 7/8 (chặn nhầm id 107, lọt id 11, 15); sau khi sửa prompt, lần 2
  attacks 20/20, clean 8/8 (false_positive_rate 0.0), output 4/4, input guard P95
  1199ms. `check_lab.py` báo PASS cho `phase_c_guard.py` và Phase C artifact.
- Điều đã học: clean controls là cần thiết vì guard luôn chặn cũng đạt 20/20
  attacks. NER tiếng Anh của Presidio gắn PERSON cho câu tiếng Việt ("Xin lỗi",
  "bao nhiêu") nên nếu chặn mọi entity thì 7/8 câu sạch bị chặn; phải giới hạn
  entity theo mục tiêu. Prompt self-check cần định nghĩa phạm vi rõ: "hỏi quy định
  mật khẩu" khác "xin mật khẩu". Latency Task 12 chỉ là input layers; gần như toàn
  bộ thời gian là lời gọi LLM của NeMo (P95 khoảng 1192ms), Presidio khoảng 14ms.
- Giới hạn/cách xử lý: prompt được chỉnh sau khi xem kết quả trên chính bộ 20/8,
  nên 28/28 là kết quả lạc quan; cần bộ held-out để đánh giá lại. P95 vượt budget
  500ms và dao động 915–1199ms giữa hai lần chạy; ghi đúng số đo, không sửa. Bộ lọc
  PII chỉ gồm ba loại entity nên có thể bỏ sót tên, địa chỉ. Đã cân nhắc đổi sang
  `gpt-6-luna` cho rẻ hơn nhưng giữ `gpt-4o-mini` vì NeMo 0.10.0 gọi self-check với
  `max_tokens=3`, không phù hợp với reasoning model.
- Nguồn/AI hỗ trợ nếu có: Claude Code (Anthropic) hỗ trợ cài môi trường, viết các
  hàm Phase C, phân tích false positive của Presidio và viết lại prompt input rail.
  Tự kiểm chứng bằng unit tests, chạy Presidio offline trên 28 inputs, hai batch
  live và `check_lab.py`; số liệu lấy trực tiếp từ `artifacts/guard_results.json`.

### TranTuanHoang-2A202602832

- Phần được giao và phần thực tế đã làm: Phụ trách toàn bộ Phase A. Tích hợp 5 module m1–m5 từ Day 18 (`m1_chunking.py`, `m2_search.py`, `m3_rerank.py`, `m4_eval.py`, `m5_enrichment.py`). Triển khai Tasks 1–4 trong `src/phase_a_ragas.py`: Task 1 `group_by_distribution` (chia đúng 20/20/10), Task 2 `run_ragas_50q` (chuẩn hóa adapter gọi `m4_eval.evaluate_ragas`, trả về `RagasEvaluation` chứa `results` và `evaluator_models`), Task 3 `bottom_10` (trích xuất 10 câu điểm thấp nhất tăng dần, map `diagnosis`/`suggested_fix`), Task 4 `cluster_analysis` (tạo ma trận worst_metric 4×3, tính dominant distribution theo mean thấp nhất, dominant metric và insight). Tinh chỉnh `m4_eval.py` sử dụng `llm_factory` và `embedding_factory` động để trích xuất model IDs chuẩn thay vì hardcode, loại bỏ `or 0.0`. Chạy sinh 50 answers và batch Phase A live; hoàn thiện báo cáo phân tích lỗi `doc/reports/failure_clusters.md`.
- Bằng chứng: PR #1 (code logic Phase A), PR #2 (`artifacts/answers_50q.json`), commit cập nhật `src/m4_eval.py`, `artifacts/ragas_50q.json` (created_at `2026-10-12T13:47:35.816200+00:00`), `doc/reports/failure_clusters.md`.
- Cách kiểm tra: Offline tests `python -m pytest tests/test_phase_a.py -q` pass toàn bộ 6/6 test cases. Sinh answers bằng `python scripts/setup_answers.py --live`. Chạy batch live bằng `python scripts/run_experiments.py --phase a --live`. Kiểm tra tính hợp lệ qua `validate_phase_a` và schema verification trong `check_lab.py`.
- Điều đã học: Hiểu rõ vai trò và giới hạn của 4 metric Ragas: faithfulness xem câu trả lời có bám context không, answer relevancy xem trả lời có đúng trọng tâm câu hỏi không, context precision/recall đo lường chất lượng retrieval. Trong multi-hop và adversarial queries, retrieval thiếu chunk (recall thấp) dẫn đến generator trả về "Không tìm thấy", kéo tụt cả faithfulness và relevancy về 0 dù LLM không hề bịa đặt. Ma trận lỗi đếm theo worst metric của mọi câu chứ không phải chỉ câu fail, và nhóm yếu nhất phải chọn theo mean avg_score thấp nhất thay vì tổng số lượng câu.
- Giới hạn/cách xử lý: Khi retrieval lấy thiếu thông tin cho câu hỏi đa chặng (multi-hop) hoặc không phân biệt được phiên bản cũ/mới của policy (adversarial), pipeline chọn phương án an toàn là từ chối trả lời ("Không tìm thấy"). Cần nâng cấp Hybrid Search kết hợp BM25 mở rộng và reranker để tăng recall cho multi-hop, đồng thời bổ sung metadata filtering theo ngày hiệu lực để loại bỏ chunk tài liệu đã hết hạn.
- Nguồn/AI hỗ trợ nếu có: Antigravity AI hỗ trợ sửa lỗi tương thích môi trường PyArrow/NumPy, gợi ý tích hợp factory cho Ragas m4, và hỗ trợ rà soát schema các báo cáo. Toàn bộ logic, kiểm chứng bằng chứng và phân tích số liệu thực tế được tự thực thi và xác nhận đối chiếu với kết quả chạy live.

### NguyenVanHuong-2A202602743

- Phần được giao: Phase B — pairwise judge, hoán đổi thứ tự, Cohen’s κ và bias
  report. Phần hiện đã triển khai với hỗ trợ Codex: Tasks 5–8 trong
  `src/phase_b_judge.py`, kiểm tra JSON/scores, tests offline bổ sung, một batch
  judge thật và phân tích `doc/reports/bias_report.md`. Giới hạn output tokens
  của pairwise/binary judge để dùng API tiết kiệm; không chạy lại batch chỉ
  để kiểm chứng file kết quả.
- Bằng chứng: các file trên; helper binary trong `src/experiment_support.py`
  chỉ bổ sung giới hạn output và reasoning ngắn, không đổi nguồn calibration.
  Live evidence: `artifacts/judge_results.json`, created_at UTC
  `2026-10-12T12:30:37.395169+00:00`. Chưa xác nhận commit nộp bài.
- Cách kiểm tra: `.venv-tests/bin/python -m pytest tests/test_phase_b.py
  tests/test_support.py tests/test_report_contracts.py -q`; kết quả 84 passed
  (24 Phase B và 60 support/report contracts), dùng SDK mock, không gọi API.
  Batch live dùng runner `--phase b --live`; JSON và metadata được validate
  offline theo schema/sources/hash/κ/bias sau khi lưu, không gọi API lại.
- Điều đã học: Pairwise A/B/tie so sánh hai câu trả lời, còn nhãn 0/1 đánh giá
  một câu trả lời theo reference; không thể lấy winner A/B làm nhãn tốt/xấu.
  Khi hoán đổi candidates, phải đổi cả winner và scores lượt 2 về A/B gốc;
  nếu hai lượt không đồng thuận thì giữ tie. Calibration phải đánh giá đúng
  model_answer gốc và không đưa human_label/human_note vào prompt để tránh
  rò rỉ nhãn. Cohen’s κ điều chỉnh mức đồng thuận ngẫu nhiên: trong batch này,
  p_o=1, p_e=0.52 nên κ=1.0, nhưng 10 nhãn khớp không bảo đảm judge đúng trên
  mọi dữ liệu hoặc reasoning luôn chính xác. Id 46 là ví dụ label đúng nhưng
  reasoning thêm điều kiện không có trong answer. Position inconsistency=0/5
  và winner dài hơn=5/5 chỉ mô tả mẫu đã chạy; cần đối chiếu nội dung và các
  cặp kiểm soát để kết luận nguyên nhân bias. Tests mock kiểm tra logic và hợp
  đồng dữ liệu; kết quả live cùng schema/source/hash mới là bằng chứng thực
  nghiệm. Giữ nguyên số liệu và kiểm chứng offline giúp tránh gọi API lặp lại.
- Giới hạn/cách xử lý: mẫu chỉ có 5 cặp và 10 nhãn; κ=1.0 không chứng minh
  judge đúng trên mọi dữ liệu. Winner dài hơn 5/5 có thể do chất lượng/độ đủ
  thông tin; reasoning id 46 thêm nhận xét không được answer hỗ trợ dù label
  khớp human. Đã giữ nguyên kết quả và phân tích các giới hạn trong report.
- Nguồn/AI hỗ trợ: Codex đọc đề và scaffold của repo, hỗ trợ triển khai, viết
  tests, kiểm chứng offline, chạy một batch thật và soạn báo cáo từ kết quả.
  Người phụ trách cần tự
  đọc, kiểm tra và xác nhận hiểu phần đóng góp trước khi nộp; unit tests không
  thay thế kết quả live hoặc tự đánh giá cá nhân.

## Cập nhật sau khi nộp

Bản nộp chính thức trước deadline; không có cập nhật bổ sung sau deadline.
