# Phân tích lỗi Phase A

Nhóm: FAMI

Người phụ trách: Trần Tuấn Hoàng — 2A202602832

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-12T13:47:35.816200+00:00.
Generation mode/fallback và điều kiện chạy: 100% câu hỏi (50/50 câu) hoàn thành qua generation_mode=llm với model gpt-4o-mini-2024-07-18, không sử dụng extractive_fallback. Điều kiện chạy thực tế trên môi trường WSL2 Ubuntu 24.04 (Python 3.11.11, PyTorch CPU, Docker Qdrant v1.9.7 localhost:6333, full pipeline Day 18 m1–m5).
Models generation/evaluator: generation: gpt-4o-mini-2024-07-18; evaluator: gpt-4o-mini, text-embedding-ada-002 (trích xuất động từ llm_factory và embedding_factory của m4_eval).

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 0.9833 | 0.4437 | 0.8000 |
| answer_relevancy | 0.8398 | 0.6681 | 0.7058 |
| context_precision | 0.9833 | 0.9917 | 0.9333 |
| context_recall | 0.9250 | 0.7958 | 0.7167 |
| avg_score | 0.9329 | 0.7248 | 0.7890 |

## Bottom 10 và nguyên nhân

| Rank | ID | Question | Distribution | Avg Score | Worst Metric | Diagnosis | Suggested Fix |
|---|---|---|---|---|---|---|---|
| 1 | 39 | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA. | multi_hop | 0.2500 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 2 | 21 | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | multi_hop | 0.3750 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 3 | 30 | So sánh quyền lợi bảo hiểm giữa nhân viên thử việc và nhân viên chính thức. | multi_hop | 0.3750 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 4 | 33 | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | multi_hop | 0.3750 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 5 | 50 | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | adversarial | 0.4167 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 6 | 48 | Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không? | adversarial | 0.6667 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 7 | 44 | Bao lâu phải đổi mật khẩu một lần? | adversarial | 0.6931 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 8 | 25 | Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu? | multi_hop | 0.7018 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 9 | 38 | Nhân viên đi công tác nước ngoài 4 ngày tại thành phố lớn, ở khách sạn 200 USD/đêm (3 đêm). Công ty thanh toán bao nhiêu tiền khách sạn? | multi_hop | 0.7045 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 10 | 31 | Nhân viên đi công tác trong nước, ở khách sạn 2 đêm với giá 1.500.000 VNĐ/đêm. Công ty thanh toán tối đa bao nhiêu cho tiền khách sạn? | multi_hop | 0.7485 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |

### Đối chiếu bằng chứng từ các câu đại diện:

- **Lỗi câu đa chặng (multi-hop retrieval bottleneck) — ID 39 và ID 21:**
  - *ID 39 (So sánh mật khẩu v1.0 và v2.0):* Reference yêu cầu đối chiếu cả hai phiên bản (v1: 8 ký tự, 90 ngày, không MFA; v2: 12 ký tự, 120 ngày, bắt buộc MFA). Contexts lấy về gồm 3 đoạn từ `mat_khau_v1.md` và `mat_khau_v2.md` nhưng chỉ chứa thông báo thay thế và phương thức MFA, thiếu hoàn toàn quy định cụ thể về độ dài và chu kỳ đổi của v1 lẫn v2. Vì thiếu bằng chứng, LLM tuân thủ system prompt nghiêm ngặt nên trả lời "Không tìm thấy.", khiến faithfulness, answer_relevancy và recall đều nhận 0.0.
  - *ID 21 (Senior 9 năm thâm niên: ngày phép và khoảng lương):* Retriever lấy được quy định thâm niên (18 ngày phép) trong `nghi_phep_nam_v2024.md` nhưng bỏ sót tài liệu `bang_luong_2024.md` (lương Senior 20-35 triệu). LLM từ chối trả lời ("Không tìm thấy."), khiến faithfulness = 0.0 dù context recall đạt 0.5.

- **Bẫy mâu thuẫn phiên bản (Version Conflict) — ID 44:**
  - *ID 44 (Bao lâu phải đổi mật khẩu một lần?):* Answer của pipeline là "120 ngày." (hoàn toàn chính xác theo policy v2.0 hiện hành). Tuy nhiên, cả 3 contexts được retrieve gồm đoạn MFA và đoạn từ `mat_khau_v1.md` (chính sách cũ 90 ngày), không đoạn nào chứa chuỗi "120 ngày". LLM đã dùng kiến thức tiềm ẩn để trả lời đúng, nhưng vì không có bằng chứng hỗ trợ trong context nên Ragas chấm faithfulness = 0.0. Đây là minh chứng rõ ràng cho việc retrieval bị nhiễu bởi phiên bản cũ hết hiệu lực.

- **Bẫy mâu thuẫn quy định và phủ định — ID 50 và ID 48:**
  - *ID 50 (Dùng VPN cá nhân khi WFH):* Câu hỏi nghe hợp lý nhưng vi phạm chính sách VPN v1.3 (cấm VPN cá nhân). Retriever chỉ lấy được hướng dẫn WireGuard mà không lấy được điều khoản cấm, dẫn đến LLM trả lời "Không tìm thấy." và faithfulness = 0.0.
  - *ID 48 (Nhân viên thử việc có hưởng bảo hiểm PVI không):* Context `thu_viec.md` có ghi "chưa được hưởng bảo hiểm sức khỏe PVI", nhưng LLM lại trả lời "Không tìm thấy.", làm answer relevancy rơi về 0.0.

## Matrix và nhóm yếu nhất

| Metric | factual (20) | multi_hop (20) | adversarial (10) | Tổng |
|---|---|---|---|---|
| faithfulness | 1 | 17 | 2 | 20 |
| context_recall | 3 | 2 | 5 | 10 |
| context_precision | 1 | 0 | 1 | 2 |
| answer_relevancy | 15 | 1 | 2 | 18 |
| Tổng | 20 | 20 | 10 | 50 |

Nhóm yếu theo mean thấp nhất: multi_hop; mean: 0.7248 (thấp hơn factual 0.9329 và adversarial 0.7890).
Worst metric xuất hiện nhiều nhất: faithfulness; số câu: 20 câu (chiếm 40% tập dữ liệu, trong đó 17 câu thuộc multi_hop).

### Giải thích nguyên nhân và giải pháp:
- **Nguyên nhân:** Nhóm multi_hop yếu nhất vì faithfulness trung bình chỉ đạt 0.4437. Căn nguyên không phải do LLM tự bịa đặt vô căn cứ, mà là do retrieval chỉ lấy `top_k=3` đoạn văn, không đủ dung lượng để chứa đầy đủ thông tin từ 2 đến 3 file văn bản độc lập (ví dụ vừa cần chính sách nghỉ phép vừa cần thang bảng lương). Do thiếu dữ liệu, system prompt ép buộc LLM trả lời "Không tìm thấy.", khiến faithfulness bị Ragas chấm 0.0. Ngược lại, nhóm factual có độ khớp cao (mean 0.9329) vì một đoạn đơn lẻ đã đủ trả lời.
- **Giải pháp đề xuất:**
  1. Tăng `top_k` của Hybrid Search từ 3 lên 5 hoặc 7 cho các câu hỏi phức hợp.
  2. Bổ sung kỹ thuật phân rã câu hỏi (Query Decomposition) trước khi truy vấn vector để tìm kiếm riêng rẽ từng thực thể.
  3. Áp dụng Cross-Encoder Reranker để chọn lọc các đoạn giàu thông tin nhất từ nhiều tài liệu khác nhau.
- **Cách kiểm chứng lại:** Chạy lại `python scripts/setup_answers.py --live` sau khi cập nhật retriever, rồi đánh giá lại bằng `python scripts/run_experiments.py --phase a --live` để kiểm tra xem mean avg_score của multi_hop có vượt ngưỡng 0.85 hay không.

## Bẫy adversarial và giới hạn

- **So sánh với factual và multi_hop:** Nhóm adversarial đạt điểm trung bình 0.7890, nằm giữa factual (0.9329) và multi_hop (0.7248). Điểm yếu tập trung vào `context_recall` (0.7167) và `answer_relevancy` (0.7058), với 5/10 câu có worst metric là context_recall.
- **Xung đột phiên bản (Version Conflict):** Trong kho tài liệu tồn tại song song cả chính sách cũ và mới (`mat_khau_v1.md` và `mat_khau_v2.md`, `nghi_phep_nam_v2023.md` và `nghi_phep_nam_v2024.md`). Do từ khóa giống hệt nhau, mô hình tìm kiếm dễ đưa các đoạn cũ vào ngữ cảnh nếu không có bộ lọc thời gian. ID 44 là ví dụ điển hình khi answer đúng phiên bản mới nhưng ngữ cảnh lại trích từ bản cũ.
- **Bẫy phủ định (Negation Traps):** Các câu hỏi như ID 48 và ID 50 kiểm tra khả năng nhận biết mệnh đề cấm hoặc ngoại lệ. Mô hình embedding hiện tại chưa phân biệt tốt các sắc thái phủ định (ví dụ: "được hưởng" so với "chưa được hưởng"), dẫn đến việc trích xuất thiếu điều khoản loại trừ.
- **Điểm thấp không đồng nghĩa với nhận diện bẫy:** Điểm số thấp ở nhóm adversarial là do hệ thống không tìm đủ ngữ cảnh hoặc trả lời an toàn "Không tìm thấy", chứ không phải vì pipeline đã chủ động phát hiện câu hỏi mang tính tấn công hay bẫy nghiệp vụ.
- **Giới hạn thực nghiệm:** Bộ dữ liệu adversarial chỉ gồm 10 câu hỏi, chưa đủ để khái quát hóa toàn diện các loại hình tấn công tinh vi (như indirect prompt injection). Mô hình generation hiện tại (gpt-4o-mini) có xu hướng từ chối trả lời khi ngữ cảnh không hoàn toàn khẳng định. Cần bổ sung metadata filtering theo ngày hiệu lực của chính sách để loại bỏ triệt để tài liệu hết hạn.
