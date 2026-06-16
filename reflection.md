# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Benchmark được chạy bằng `BenchmarkRunner` của Day14 trên 20 QA pairs về Luật Phòng, chống ma túy/Điều 29, với agent là RAG thật từ Day8 (`generate_with_citation`).

**Overall pass rate:** 50.0%

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.74 | 0.66 | 0.81 | 0.03 |
| Relevance | 0.68 | 0.29 | 0.92 | 0.15 |
| Completeness | 0.45 | 0.00 | 1.00 | 0.29 |
| Overall Score | 0.62 | 0.40 | 0.88 | 0.13 |

**Score interpretation (theo bài giảng):**
- Bao nhiêu metrics ở Good (0.8-1.0)? 8
- Bao nhiêu metrics ở Needs Work (0.6-0.8)? 34
- Bao nhiêu metrics ở Significant Issues (<0.6)? 18

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 0 | 0% |
| irrelevant | 1 | 10% |
| incomplete | 5 | 50% |
| off_topic | 4 | 40% |
| refusal | 0 | 0% |

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

Theo bài giảng: "Phân loại failure TRƯỚC KHI fix. Đừng fix từng failure riêng lẻ — CLUSTER rồi fix root cause."

### Failure 1

**Question:** *Một người đã hoàn thành điều trị và giáo dục nhưng chưa chuẩn bị tái hòa nhập cộng đồng. Người đó đã hoàn thành quy trình cai nghiện bắt buộc chưa?*

**Agent Answer:** *RAG trích đúng Điều 29 và liệt kê các giai đoạn, nhưng không kết luận trực tiếp "chưa hoàn thành"; cuối câu còn nói cần kết luận chi tiết hơn thì không thể xác minh.*

**Scores:** Faithfulness: 0.73 | Relevance: 0.46 | Completeness: 0.00 | Overall: 0.40

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Câu trả lời có context đúng nhưng không trả lời trực tiếp "chưa hoàn thành". |
| Why 1 | Tại sao xảy ra? | Generator fallback chỉ tóm tắt chunk liên quan nhất thay vì suy luận từ Điều 29. |
| Why 2 | Tại sao Why 1 xảy ra? | Prompt/fallback chưa bắt buộc đưa kết luận ngắn trước khi trích dẫn. |
| Why 3 | Tại sao Why 2 xảy ra? | Pipeline ưu tiên faithfulness, tránh suy đoán, nhưng thiếu rule cho câu hỏi yes/no pháp luật. |
| Why 4 | Root cause là gì? | Generation layer thiếu answer template cho câu hỏi điều kiện/hoàn thành quy trình. |

**Root cause (from `find_root_cause()`):**
> Answer is missing key information — increase context window or improve generation

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Có. Retrieval đã lấy đúng Điều 29, nên lỗi chính không phải thiếu context mà là generation chưa chuyển evidence thành kết luận pháp lý rõ ràng.

**Proposed fix (cụ thể, actionable):**
> Thêm prompt rule: với câu hỏi "đã hoàn thành chưa/có hợp lệ không", phải trả lời trực tiếp "Có/Không/Chưa" ở câu đầu, sau đó mới giải thích và cite Điều 29.

---

### Failure 2

**Question:** *Một cơ sở chỉ thực hiện điều trị cắt cơn và học nghề nhưng bỏ qua giáo dục phục hồi hành vi, quy trình có hợp lệ không?*

**Agent Answer:** *RAG trích Điều 29 và các giai đoạn nhưng không nêu kết luận "không hợp lệ"; answer bị cắt ở phần "lao động trị liệu, học nghề" và thiếu phần phân tích rằng giai đoạn giáo dục phục hồi là bắt buộc.*

**Scores:** Faithfulness: 0.73 | Relevance: 0.64 | Completeness: 0.00 | Overall: 0.46

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Answer thiếu kết luận pháp lý "không hợp lệ". |
| Why 1 | Tại sao xảy ra? | RAG chỉ copy/tóm tắt đoạn quy trình thay vì so sánh các bước đã làm với các bước bắt buộc. |
| Why 2 | Tại sao Why 1 xảy ra? | Không có reasoning checklist để kiểm tra từng giai đoạn a, b, c, d, đ. |
| Why 3 | Tại sao Why 2 xảy ra? | Prompt chưa yêu cầu "đối chiếu facts trong câu hỏi với điều kiện trong luật". |
| Why 4 | Root cause là gì? | Thiếu rule cho multi-step legal reasoning trên quy trình Điều 29. |

**Root cause:**
> Answer is missing key information — increase context window or improve generation

**Proposed fix:**
> Tạo few-shot example cho dạng câu hỏi "bỏ qua một giai đoạn"; yêu cầu model liệt kê giai đoạn bị thiếu và kết luận quy trình không hợp lệ nếu thiếu bước bắt buộc.

---

### Failure 3

**Question:** *Một người hoàn thành 4 giai đoạn đầu nhưng chưa tái hòa nhập cộng đồng, có được coi là hoàn thành cai nghiện bắt buộc không?*

**Agent Answer:** *RAG lấy đúng đoạn Điều 29 khoản 2: cai nghiện bắt buộc phải bảo đảm đầy đủ các giai đoạn, nhưng answer vẫn không mở đầu bằng kết luận "Không".*

**Scores:** Faithfulness: 0.72 | Relevance: 0.67 | Completeness: 0.00 | Overall: 0.46

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Có evidence đúng nhưng thiếu expected answer ngắn "Không". |
| Why 1 | Tại sao xảy ra? | Fallback answer chỉ nói "thông tin có thể kiểm chứng là..." rồi trích đoạn. |
| Why 2 | Tại sao Why 1 xảy ra? | Fallback không thực hiện bước conclusion extraction từ retrieved context. |
| Why 3 | Tại sao Why 2 xảy ra? | RAG generation phụ thuộc vào LLM API; khi không có API thì fallback quá thô. |
| Why 4 | Root cause là gì? | Cần cải thiện fallback generator hoặc đảm bảo LLM generation hoạt động trong benchmark. |

**Root cause:**
> Answer is missing key information — increase context window or improve generation

**Proposed fix:**
> Nâng cấp fallback answer: nếu query chứa "có được coi", "đã hoàn thành", "có hợp lệ", thì dùng rule-based conclusion dựa trên token như "phải bảo đảm đầy đủ các giai đoạn" để trả lời trực tiếp.

---

## 3. Failure Clustering

Theo bài giảng: "Fix 1 root cause giải quyết nhiều failures cùng lúc."

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|------------|--------------------:|----------|
| 1 | Generation thiếu kết luận trực tiếp cho câu hỏi yes/no hoặc hợp lệ/không hợp lệ | 5 | High |
| 2 | Answer có liên quan nhưng chưa bám đúng intent câu hỏi | 4 | Medium |
| 3 | Out-of-scope/adversarial chưa được route sang refusal/clarification rõ ràng | 1 | Medium |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> Chọn Cluster 1 vì chiếm nhiều failure nhất và ảnh hưởng trực tiếp đến completeness. Retrieval đã lấy đúng Điều 29, nên cải thiện generation template có thể tăng điểm cho nhiều câu cùng lúc mà không cần thay đổi toàn bộ retriever.

---

## 4. Improvement Log (from `generate_improvement_log`)

Paste output của `generate_improvement_log()`:

```markdown
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | incomplete | Answer is missing key information — increase context window or improve generation | Add direct yes/no conclusion template for legal procedure questions | Open |
| F002 | incomplete | Answer is missing key information — increase context window or improve generation | Add few-shot examples for skipped-stage reasoning under Article 29 | Open |
| F003 | incomplete | Answer is missing key information — increase context window or improve generation | Improve fallback generator to extract a legal conclusion before citation | Open |
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. Add few-shot examples and answer checklists to improve completeness.
2. Improve prompt clarity and intent routing so answers directly address the user question.
3. Track recurring failures in the golden dataset and rerun regression tests before deploy.

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Câu 1: Khi nào chạy `run_regression()` trong production system?**
> Chạy trước mỗi merge vào `main`, sau mỗi thay đổi prompt/retriever/chunking/reranking, trước demo hoặc release, và sau khi thêm golden dataset mới từ các failure thực tế.

**Câu 2: Threshold regression 0.05 có phù hợp domain của bạn không?**
> Với domain pháp luật, 0.05 là mức hợp lý cho CI vì chỉ cần giảm nhẹ faithfulness hoặc completeness cũng có thể làm câu trả lời gây hiểu sai. Với release quan trọng, có thể strict hơn cho faithfulness, ví dụ block nếu giảm hơn 0.03.

**Câu 3: Khi phát hiện regression — block deployment hay chỉ alert?**
> Nên block deployment nếu regression thuộc faithfulness hoặc completeness, vì đây là domain pháp luật cần căn cứ và đủ điều kiện. Với relevance giảm nhẹ trên câu adversarial, có thể alert để review thủ công nếu không ảnh hưởng nhóm câu core.

**Câu 4: Eval pipeline nên chạy ở đâu trong CI/CD flow?**

```
Code change → [Unit tests] → [Offline RAG benchmark + run_regression] → [Manual review worst failures] → Deploy
              (bước 1)      (bước 2)                           (bước 3)
```

---

## 6. Continuous Improvement Loop

Theo bài giảng: Evaluate → Analyze → Improve → Augment (add to benchmark) → lặp lại

**Sau lab hôm nay, 3 actions tiếp theo bạn sẽ làm để improve agent:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
| 1 | Thêm answer template bắt buộc kết luận trực tiếp trước citation cho câu hỏi yes/no pháp luật | Completeness, Relevance | Tăng điểm các case M04, H01, H05 |
| 2 | Thêm few-shot examples cho Điều 29: đủ 5 giai đoạn, thiếu giai đoạn, tự nguyện vs bắt buộc | Completeness | Giảm lỗi thiếu kết luận và thiếu điều kiện |
| 3 | Thêm intent classifier cho out-of-scope/adversarial trước khi gọi RAG | Relevance, Safety | A01/A02/A03 được xử lý rõ hơn |

**Bạn sẽ thêm failure cases nào vào benchmark cho sprint tiếp theo?**
> Thêm các case hỏi "đã hoàn thành chưa", "quy trình có hợp lệ không", "tự nguyện có bắt buộc học nghề không", và các câu out-of-scope có keyword gây nhiễu như "giá", "đầu tư", "che giấu".

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** RAGAS-inspired heuristic

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**
> Tôi sẽ dùng kết hợp heuristic custom cho CI nhanh và DeepEval/RAGAS cho evaluation định kỳ. Heuristic rẻ, deterministic, dễ debug; DeepEval/RAGAS tốt hơn để đánh giá ngữ nghĩa, citation, faithfulness và các câu paraphrase mà word-overlap có thể chấm sai.

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... | RAGAS/DeepEval có faithfulness, answer relevancy, context recall/precision phù hợp trực tiếp với RAG pháp luật. |
| CI/CD integration vì... | Có thể chạy threshold-based quality gate; heuristic chạy mỗi commit, DeepEval/RAGAS chạy trước release. |
| Team workflow vì... | Kết quả dạng bảng giúp cả team nhìn thấy failure type, worst cases và root cause để cải thiện prompt/retrieval/generation. |
