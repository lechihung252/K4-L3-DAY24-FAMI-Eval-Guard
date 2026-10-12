# Blueprint CI/CD cho RAG và guardrails

Nhóm: FAMI

Người phụ trách: Lê Chí Hùng — 2A202602863 (tích hợp), dùng kết quả A/B/C của cả nhóm

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: artifacts/ragas_50q.json, judge_results.json, guard_results.json.
Run timestamp: guard 2026-10-12T10:06:35Z, judge 2026-10-12T12:30:37Z,
ragas 2026-10-12T13:47:35Z; guard models: `gpt-4o-mini`.
Models generation/evaluator/judge từ A/B: generation `gpt-4o-mini-2024-07-18`
(answers_50q.json, ghi lại trong judge_results.json); judge `gpt-4o-mini`
(judge_results.json); evaluator `gpt-4o-mini`, `text-embedding-ada-002`
(ragas_50q.json, lấy từ m4).

## Kiến trúc và failure action

Input → Presidio → NeMo input → RAG → NeMo output → response.
Request thực dừng sớm khi bị chặn; khác phép đo input layers chạy đủ hai tầng.

Luồng dữ liệu và hành động khi lỗi:

| Tầng | Kiểm tra | Khi phát hiện | Khi chính tầng đó lỗi |
|---|---|---|---|
| Presidio | `VN_CCCD`, `VN_PHONE`, `EMAIL_ADDRESS` trong input | Chặn, trả câu từ chối; log bản đã anonymize | Fail-closed: từ chối và báo lỗi, không chuyển input thô sang LLM |
| NeMo input | Off-topic, jailbreak, prompt injection, xin dữ liệu cá nhân/mật khẩu | Trả câu từ chối trong `rails.co` | Timeout/API lỗi: fail-closed, trả câu từ chối chung, tăng counter lỗi |
| RAG | Hybrid search, rerank, generation | Không áp dụng | Không có contexts: trả "chưa tìm thấy chính sách", không để LLM tự trả lời |
| NeMo output | CCCD/CMND, SĐT/email riêng, lương cá nhân, mật khẩu trong answer | Thay answer bằng câu từ chối | Fail-closed: không trả answer chưa kiểm tra |

Quyết định chặn đọc từ `activated_rails[].stop` của NeMo, không dò từ khóa "xin lỗi"
vì câu sạch id 102 cũng chứa cụm này. Log chỉ lưu text đã qua Presidio anonymize,
không lưu input thô. API key nằm trong `.env` khi chạy local và trong secret store
của CI khi triển khai; không ghi vào code, log hay artifacts. Email helpdesk công
khai `helpdesk@cty.vn` được phép trong output (output id 202 giữ nguyên).

## Phạm vi và số đo latency

Bắt buộc Task 12: **input_layers_sequential**, Presidio + NeMo input, không
short-circuit. Engines tạo trước đo; có hai warm-up thật không tính vào mẫu.
Percentile nội suy tuyến tính từ raw samples; tổng từ tổng từng sample.

Số mẫu timed: 20; warm-up: 2;
phương pháp: linear.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 7.7 | 14.0 | 17.8 | Đã đo Task 12; không phải bottleneck |
| NeMo input | 650.2 | 1192.1 | 1282.8 | Đã đo Task 12; bottleneck, một lời gọi LLM mỗi input |
| Input guard sequential | 659.0 | 1199.3 | 1289.8 | Đã đo Task 12; vượt target 500ms |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms |

Input-layer budget target: 500ms; kết quả thực tế P95 1199.3ms,
`latency_budget_ok = false`.

Bottleneck là NeMo input: Presidio chỉ chiếm khoảng 1% thời gian. Mẫu NeMo nhanh
nhất là 543.6ms, nên không mẫu nào đạt target dù chỉ đo riêng tầng này; nguyên
nhân là self-check gọi `gpt-4o-mini` qua mạng cho mỗi input. Lần chạy trước khi
sửa prompt đo P95 915ms, cho thấy số đo dao động mạnh theo mạng/API với 20 mẫu.

Kế hoạch đo phần ngoài phạm vi: thêm timestamp `perf_counter` quanh RAG
(search, rerank, generation) và NeMo output trong cùng event loop, giữ 2 warm-up,
lấy ít nhất 50 mẫu từ 8 câu sạch và 20 attacks, ghi raw samples. Full request đo
riêng hai nhóm: request bị chặn sớm (chỉ input layers) và request đi hết pipeline.

## CI gates đề xuất

- RAGAS faithfulness ≥0.75, avg_score ≥0.65 trên 50 câu.
- Attacks ≥18/20 cho CI đề xuất; clean 8/8 allowed; output cases đúng.
- Full guard P95 <500ms chỉ áp dụng sau khi thực sự đo full guard.

Mốc bắt buộc lab vẫn attacks ≥15/20; bonus ≥18/20 kèm clean controls đạt.

Job đề xuất, chạy khi đổi `src/`, `guardrails/` hoặc `fixtures/`:

```bash
python -m pytest tests/ -q                  # offline, mock; exit != 0 khi test fail
python scripts/run_experiments.py --phase c --live --latency-samples 20
python scripts/check_lab.py                 # exit 1 khi có check fail
```

`check_lab.py` hiện đã trả exit 1 khi attacks dưới 15/20, có câu sạch bị chặn,
output case sai hoặc artifacts lệch schema/hash, nên dùng được làm gate tối thiểu.
Các ngưỡng cao hơn (attacks ≥18/20, faithfulness ≥0.75, avg_score ≥0.65) cần thêm
một script gate đọc JSON và so với biến `MIN_ATTACKS`, `MIN_FAITHFULNESS`,
`MIN_AVG_SCORE`; script này **chưa được viết**, các biến MIN_* chỉ là đề xuất.
Áp vào kết quả hiện tại: avg_score 0.821 qua ngưỡng 0.65, nhưng faithfulness
0.731 **không qua** ngưỡng 0.75, nên pipeline hiện tại sẽ bị gate chặn cho tới khi
sửa nhóm multi_hop (xem cải tiến 5).
Latency chưa nên làm gate cứng vì dao động 915–1199ms giữa hai lần chạy; trước
mắt chỉ cảnh báo khi P95 input guard tăng quá 20% so với lần chạy trước.
Workflow CI chưa được triển khai trong repo này.

## Monitoring và cải tiến

| Metric | Ngưỡng cảnh báo | Cách lấy | Người xử lý |
|---|---|---|---|
| Tỷ lệ chặn theo tầng/loại | Thay đổi quá 2 lần so với trung bình 7 ngày | Đếm `blocked_by` và loại rail trong log | Người phụ trách guard |
| False positive | Bất kỳ khiếu nại "bị chặn nhầm" đã xác nhận | Kênh báo lỗi của người dùng và review ngẫu nhiên 20 request bị chặn mỗi tuần | Người phụ trách guard |
| P95 input guard và lỗi/timeout NeMo | P95 vượt 1500ms hoặc lỗi vượt 1% | Raw latency mỗi request, counter lỗi | Người tích hợp |
| PII trong output | Bất kỳ output nào bị output rail chặn vì PII | Log output rail (đã anonymize) | Người phụ trách guard |
| Chất lượng RAG/judge | faithfulness dưới 0.75; κ dưới 0.6 | Chạy lại RAGAS 50 câu và 10 nhãn calibration mỗi tháng hoặc khi đổi model/prompt | Phase A/B |

Drift: mỗi tháng lấy mẫu câu hỏi thật (đã anonymize) để bổ sung bộ test, vì
corpus có nhiều phiên bản chính sách (nghỉ phép v2023/v2024, mật khẩu v1/v2).
Privacy: chỉ dữ liệu đã anonymize được đưa vào bộ mẫu và công cụ review.

RAGAS mean/worst distribution: trên 50 câu faithfulness 0.731, answer_relevancy
0.744, context_precision 0.977, context_recall 0.832, avg_score 0.821. Nhóm yếu
nhất theo mean là multi_hop (0.725), sau đó adversarial (0.789), factual (0.933);
faithfulness là worst metric của 20/50 câu, trong đó 17 câu multi_hop.
Cohen’s κ: 1.0 (10/10 nhãn judge khớp nhãn người; position bias 0/5,
verbosity win rate 5/5).
Attacks passed: 20/20; clean false_positive_rate: 0.0.

Giới hạn: bộ 20 attacks/8 câu sạch/4 outputs nhỏ, và prompt `self_check_input`
đã được chỉnh sau khi xem kết quả lần 1 (18/20 attacks, 7/8 câu sạch) trên chính
bộ này, nên 20/20 và 8/8 là kết quả lạc quan. Calibration chỉ có 10 nhãn và 5 cặp
pairwise nên κ=1.0 và verbosity 5/5 chưa đủ để kết luận về judge.

Cải tiến có thể kiểm tra:

1. Thay self-check LLM ở tầng input bằng classifier chạy cục bộ, hoặc chạy NeMo
   input song song với retrieval; đo lại bằng đúng phương pháp Task 12 và kỳ vọng
   P95 input guard dưới 500ms.
2. Tạo bộ held-out mới (attacks và câu sạch chưa dùng để chỉnh prompt), chạy lại
   suite để đo tỷ lệ chặn và false positive không bị ảnh hưởng bởi việc tuning.
3. Mở rộng Presidio với NER hỗ trợ tiếng Việt cho tên/địa chỉ; kiểm tra rằng
   false_positive_rate trên câu sạch vẫn bằng 0 (NER tiếng Anh hiện gắn PERSON
   sai cho 7/8 câu sạch nên đang bị loại).
4. Thêm cặp pairwise kiểm soát độ dài (cùng nội dung, khác độ dài) để kiểm tra
   verbosity win rate 5/5 là do độ dài hay do câu dài hơn thực sự đúng/đủ hơn.
5. Nhóm multi_hop có faithfulness trung bình 0.444 trong khi context_precision
   0.992, nên lỗi chủ yếu ở bước tổng hợp nhiều đoạn hoặc ở chỗ retrieval thiếu
   một chặng (context_recall 0.796) khiến answer bị đánh giá thiếu căn cứ. Đọc lại
   7 câu multi_hop trong bottom 10, tăng số chunk đưa vào generation cho câu hỏi
   đa chặng, chạy lại Phase A và kỳ vọng faithfulness multi_hop tăng, tổng
   faithfulness đạt ngưỡng 0.75.
