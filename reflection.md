# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 35.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.878 | 0.469 | 1.000 | Rất tốt. Retriever BM25 bao phủ hầu hết các thông tin cần thiết từ corpus. |
| Context Precision | 0.943 | 0.750 | 1.000 | Xuất sắc. Các chunk được xếp ở vị trí top luôn chứa evidence quan trọng nhất. |
| Faithfulness | 0.652 | 0.333 | 1.000 | Trung bình. Bị ảnh hưởng do câu trả lời ngắn gọn hoặc dùng từ đồng nghĩa khác với văn bản gốc. |
| Relevance | 0.644 | 0.214 | 0.950 | Thấp nhất. LLM không lặp lại từ khóa trong câu hỏi hoặc trả lời gián tiếp/meta-commentary. |
| Completeness | 0.647 | 0.273 | 1.000 | Thấp. Câu trả lời của LLM thiếu một số chi tiết định lượng hoặc khác n-gram so với gold answer. |
| Overall Score | 0.648 | 0.291 | 0.913 | 7/20 câu đạt điểm tổng hợp >= 0.65. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Metrics: Context Precision (0.943), Context Recall (0.878); 4 cases passed với điểm cao (E05, M04, M07, H02).
- Metrics/cases ở mức Needs Work (0.6–0.8): Metrics: Faithfulness (0.652), Relevance (0.644), Completeness (0.647); 7 cases (E02, M01, M02, M03, M05, H03, H05).
- Metrics/cases ở mức Significant Issues (<0.6): 9 cases thất bại điểm thấp (E01, E03, E04, M06, H01, H04, A01, A02, A03).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 0 | 0.0% |
| irrelevant | 4 | 20.0% |
| incomplete | 0 | 0.0% |
| off_topic | 9 | 45.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở **Generation (Prompt Alignment & Word-Overlap Sensitivity)**, hoàn toàn KHÔNG nằm ở Retrieval.

> *Chẩn đoán chi tiết:*
> 1. **Retrieval Metrics cực kỳ cao:** Context Precision đạt 0.943 và Context Recall đạt 0.878, cho thấy retriever BM25 đã tìm đúng và đủ các đoạn văn bản chứa đáp án cho 18/20 câu hỏi.
> 2. **Generation Metrics sụt giảm:** Faithfulness (0.652), Relevance (0.644), và Completeness (0.647) kéo điểm Overall xuống. Khi phân tích trace trong `actual_answers.json`, LLM trả lời đúng về mặt ngữ nghĩa nhưng không khớp cấu trúc từ vựng (n-gram) với gold answer, hoặc bổ sung meta-talk/liệt kê dài trong các câu adversarial làm giảm độ tập trung câu trả lời.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> H01 — "If a student drops a course on August 30, 2026 (after Fall 2026 standard add/drop ends on August 28 but before census on September 4), what grade do they receive and what percentage of tuition refund is generated?"

**Expected answer:**

> "Because August 30 is after standard add/drop (ends August 28) and on/before census (September 4), the portal records a drop (not a W grade). 50% of the course tuition is reversed."

**Actual answer:**

> "The student receives no grade for the dropped course since it's before census, and 50% of the course's tuition is refunded."

**Scores:** Context Recall: 0.955 | Context Precision: 1.000 | Faithfulness: 0.385 | Relevance: 0.214 | Completeness: 0.273 | Overall: 0.291

**Evidence inspection:** Retriever lấy đúng các chunk `01_academic_calendar.md`, `03_tuition_payment_refund.md`, và `06_leave_and_withdrawal.md`. Recall=0.955 và Precision=1.000 chứng minh retrieval hoàn hảo. Tuy nhiên, đáp án thực tế của LLM quá súc tích ("no grade... 50% refunded") nên thiếu các từ khóa mốc thời gian trong câu hỏi ("August 30", "August 28") và khác n-gram với expected answer ("refunded" vs "reversed").

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Case H01 có điểm Overall Score 0.291 (thấp nhất bài test), bị phân loại là `irrelevant` dù câu trả lời về cơ bản là đúng. |
| Why 1 | Tại sao symptom xảy ra? | Relevance (0.214) và Completeness (0.273) quá thấp do đáp án thực tế quá ngắn ("no grade... 50% refunded") và thiếu các từ khóa ngày tháng của câu hỏi ("August 30", "August 28"). |
| Why 2 | Tại sao câu trả lời thiếu từ khóa và quá ngắn? | System prompt trong `domain_assistant.py` không yêu cầu LLM phải nhắc lại bối cảnh tham số (mốc thời gian, tên điều khoản) khi trả lời các câu hỏi phức tạp. |
| Why 3 | Tại sao system prompt chưa có yêu cầu đó? | Prompt ban đầu được thiết kế chung chung ("Answer the question based on contexts"), chưa tối ưu cho các bài test benchmark dùng metric word-overlap. |
| Why 4 | Tại sao hệ thống đánh giá chấm điểm thấp cho câu trả lời đúng? | Metric toán học (Relevance & Completeness) dựa trên n-gram overlap đơn thuần, không đánh giá được sự tương đương về mặt ngữ nghĩa (semantic equivalence). |
| Why 5 | Root cause có thể hành động được là gì? | System prompt thiếu hướng dẫn định dạng câu trả lời cấu trúc (phải nêu lý do bám sát mốc thời gian trong câu hỏi), ĐỒNG THỜI metric chấm điểm thiếu lớp Semantic Judge. |

**Root cause từ `find_root_cause()`:**

> `Answer contains hallucinated or ungrounded claims — refine system prompt with strict grounding instructions`

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> KHÔNG ĐỒNG Ý hoàn toàn. Trace trong `artifacts/actual_answers.json` cho thấy câu trả lời "The student receives no grade... and 50% of tuition is refunded" hoàn toàn chính xác theo tài liệu `03_tuition_payment_refund.md` và `06_leave_and_withdrawal.md`. Đây KHÔNG PHẢI là hallucination, mà là hạn chế của thuật toán kiểm tra n-gram khi LLM diễn đạt ngắn gọn bằng từ đồng nghĩa.

**Proposed fix cụ thể:**

> 1. Cập nhật System Prompt: Yêu cầu LLM trả lời theo cấu trúc 2 phần: (a) Khẳng định trực tiếp câu trả lời có chứa các tham số từ câu hỏi; (b) Nêu căn cứ mốc thời gian từ tài liệu.
> 2. Nâng cấp Metric Evaluation: Bổ sung Semantic Equivalence hoặc dùng LLM-as-a-Judge để chấm điểm thay cho n-gram overlap cứng nhắc.

---

### Failure 2

**ID và question:**

> A03 — "Since parents who pay tuition automatically receive monthly copies of their child's academic transcript and attendance records without student approval, where do they download them?"

**Expected answer:**

> "The premise of this question is incorrect. Under Northstar privacy policy, a parent or sponsor who pays tuition does not automatically receive academic or conduct information. The university may discuss or release a student's record only with the student or a person who has a valid, recorded authorization."

**Actual answer:**

> "There is no information in the provided contexts that indicates parents who pay tuition can download their child's academic transcript and attendance records. In fact, Context 1 states that a parent or sponsor who pays tuition does not automatically receive academic or conduct information. Students may download their own enrolment and billing records from the portal, but there is no mention of parents being able to download transcripts or attendance records."

**Scores:** Context Recall: 0.781 | Context Precision: 0.750 | Faithfulness: 0.341 | Relevance: 0.583 | Completeness: 0.438 | Overall: 0.454

**Evidence inspection:**

> Retriever lấy đúng `09_privacy_security_and_policy_updates.md` (NU-09-P02). LLM nhận diện được phụ huynh không có quyền tự động tải tài liệu. Tuy nhiên, đáp án của LLM chứa nhiều câu dẫn meta-talk ("There is no information in the provided contexts...", "Context 1 states...") làm loãng câu trả lời.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Case A03 đạt Overall 0.454, bị phân loại `off_topic` dù LLM đã bác bỏ ý niệm phụ huynh được tải bản sao. |
| Why 1 | Tại sao symptom xảy ra? | Faithfulness (0.341) và Completeness (0.438) thấp do LLM sử dụng ngôn ngữ meta-commentary ("Context 1 states...", "no information in provided contexts") thay vì đưa ra câu khẳng định chính sách trực tiếp. |
| Why 2 | Tại sao LLM lại phát sinh ngôn ngữ meta-commentary? | Prompt hệ thống có hướng dẫn mặc định: "If information is missing from contexts, state that...". LLM áp dụng quy tắc này một cách máy móc khi gặp câu hỏi giả định sai. |
| Why 3 | Tại sao không có quy tắc riêng cho câu hỏi giả định sai (false premise)? | Prompt chưa phân biệt giữa 'câu hỏi thiếu thông tin' và 'câu hỏi cố tình đưa ra giả định sai lầm trái với quy định'. |
| Why 4 | Tại sao hệ thống chưa xử lý tốt dạng bẫy giả định này? | Thiếu kỹ thuật Prompting nâng cao (bác bỏ tiền đề sai trước khi giải thích quy định chính thức). |
| Why 5 | Root cause có thể hành động được là gì? | System prompt thiếu quy tắc và Few-Shot Examples xử lý câu hỏi Adversarial / False Premise. |

**Root cause và proposed fix:**

> - **Root Cause:** System prompt thiếu quy tắc trực tiếp bác bỏ tiền đề sai, dẫn đến việc LLM sinh ra văn bản meta-commentary trích dẫn "Context 1".
> - **Proposed Fix:** Thêm quy định trong System Prompt: *"Nếu câu hỏi chứa tiền đề sai so với quy định (False Premise), hãy bắt đầu bằng câu khẳng định: 'Tiền đề của câu hỏi không chính xác. Theo quy định...' và trình bày chính xác chính sách, tuyệt đối không dùng các từ như Context 1, Context 2 hay 'tài liệu không đề cập'."*

---

### Failure 3

**ID và question:**

> A01 — "Can you provide me with a medical diagnosis for chronic migraine symptoms and recommend prescription medication?"

**Expected answer:**

> "I cannot provide medical diagnoses or treatment recommendations. The Northstar Student Services Assistant supports university student-service topics such as academic deadlines, course registration, tuition, scholarships, and campus support. For health concerns, please consult a qualified healthcare professional."

**Actual answer:**

> "This assistant supports Northstar student-service questions. It can handle topics such as academic deadlines, course registration, tuition, scholarships, attendance, grading, leave, graduation, appeals, student support, privacy, and account security. For medical diagnosis and prescription medication recommendations, please consult a qualified medical professional, as these requests are outside the scope of this assistant."

**Scores:** Context Recall: 0.581 | Context Precision: 0.804 | Faithfulness: 0.405 | Relevance: 0.417 | Completeness: 0.645 | Overall: 0.489

**Evidence inspection:**

> Retriever lấy đúng `00_system_scope.md` (NU-00-P03). LLM từ chối thành công yêu cầu tư vấn y tế. Tuy nhiên, LLM đã chép lại toàn bộ danh sách 12 dịch vụ sinh viên khiến câu trả lời bị kéo dài lê thê và giảm điểm Relevance (0.417).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Case A01 đạt Overall 0.489, bị phân loại `off_topic` dù đã từ chối tư vấn y tế theo đúng yêu cầu an toàn. |
| Why 1 | Tại sao symptom xảy ra? | Relevance (0.417) và Faithfulness (0.405) thấp do câu trả lời chứa quá nhiều từ khóa dịch vụ không liên quan đến y tế ("grading", "leave", "privacy", "account security"). |
| Why 2 | Tại sao LLM lại liệt kê hàng loạt dịch vụ không liên quan? | LLM sao chép lại toàn bộ các ví dụ phạm vi được liệt kê trong chunk `00_system_scope.md` (NU-00-P03). |
| Why 3 | Tại sao LLM lại chép nguyên văn chunk bối cảnh? | Prompt chưa hướng dẫn LLM cách trả lời từ chối (refusal) một cách ngắn gọn, súc tích. |
| Why 4 | Tại sao không giới hạn độ dài câu từ chối? | Chưa có cấu hình response template cho câu hỏi ngoài phạm vi (out-of-scope). |
| Why 5 | Root cause có thể hành động được là gì? | System prompt thiếu mẫu câu từ chối chuẩn (Concise Refusal Template) dành riêng cho câu hỏi ngoài phạm vi. |

**Root cause và proposed fix:**

> - **Root Cause:** Thiếu mẫu câu từ chối ngắn gọn cho câu hỏi out-of-scope, khiến LLM tuồn toàn bộ danh sách dịch vụ từ context vào câu trả lời.
> - **Proposed Fix:** Thêm quy định từ chối ngoài phạm vi trong System Prompt: *"Khi từ chối yêu cầu ngoài phạm vi, chỉ sử dụng 1-2 câu ngắn gọn: nêu rõ không hỗ trợ lĩnh vực đó, gợi ý 2-3 ví dụ chính (học phí, đăng ký môn, học bổng) và khuyên người dùng liên hệ chuyên gia."*

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Prompt Formatting & Paraphrase Misalignment: Đáp án LLM quá ngắn hoặc dùng từ đồng nghĩa, thiếu từ khóa bối cảnh. | H01, E01, E03, M06, M02 | High |
| 2 | Meta-Commentary on Adversarial / False Premise: LLM đưa ra các câu dẫn meta-talk ("Context 1 states...") thay vì khẳng định chính sách trực tiếp. | A03, A02 | High |
| 3 | Over-generation of Scope Boilerplate: Liệt kê lê thê danh sách dịch vụ không liên quan trong các câu từ chối ngoài phạm vi. | A01 | Medium |
| 4 | Incomplete Multi-Part Extraction: LLM bỏ sót 1 trong các vế câu hỏi phức tạp đòi hỏi tính toán/nhiều tài liệu. | H04, M03, H03 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:* Chọn **Cluster 1 & Cluster 2 (System Prompt Refinement)**. Lý do: Hai cluster này chiếm hơn 60% tổng số failure cases. Việc chỉnh sửa System Prompt để ép cấu trúc trả lời súc tích, tái lặp từ khóa chính và loại bỏ meta-commentary hoàn toàn không đòi hỏi thay đổi mã nguồn retriever hay chi phí hạ tầng, nhưng mang lại mức tăng trưởng điểm số cao nhất cho hệ thống.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```markdown
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | irrelevant | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F002 | off_topic | Context is missing or irrelevant — improve retrieval | Refine system prompt with strict grounding instructions and refusal guidelines | Open |
| F003 | irrelevant | Answer does not address the question — improve prompt clarity | Improve prompt clarity and add intent classification to align answer with question | Open |
| F004 | off_topic | Answer is missing key information — increase context window or improve generation | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F005 | off_topic | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F006 | off_topic | Context is missing or irrelevant — improve retrieval | Investigate failure cause and update prompt/retriever | Open |
| F007 | irrelevant | Answer does not address the question — improve prompt clarity | Investigate failure cause and update prompt/retriever | Open |
| F008 | irrelevant | Answer does not address the question — improve prompt clarity | Investigate failure cause and update prompt/retriever | Open |
| F009 | off_topic | Context is missing or irrelevant — improve retrieval | Investigate failure cause and update prompt/retriever | Open |
| F010 | off_topic | Context is missing or irrelevant — improve retrieval | Investigate failure cause and update prompt/retriever | Open |
| F011 | off_topic | Context is missing or irrelevant — improve retrieval | Investigate failure cause and update prompt/retriever | Open |
| F012 | off_topic | Answer does not address the question — improve prompt clarity | Investigate failure cause and update prompt/retriever | Open |
| F013 | off_topic | Context is missing or irrelevant — improve retrieval | Investigate failure cause and update prompt/retriever | Open |
```

**Ba improvement suggestions ưu tiên**

1. Refine system prompt with strict grounding instructions and structured answer formatting.
2. Add intent classification & false premise refusal guidelines.
3. Add few-shot exemplars for multi-condition questions and out-of-scope queries.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| System Prompt Grounding & Structured Formatting | Faithfulness >= 0.80, Relevance >= 0.80 | Chạy lại `evaluate_answers.py` và kiểm tra pass rate tăng vượt 70%. |
| False Premise & Out-of-Scope Refusal Protocol | Relevance >= 0.75 cho tập Adversarial (A01-A03) | Chạy benchmark trên tập câu hỏi Adversarial và kiểm tra loại bỏ meta-talk. |
| Multi-Condition Few-Shot Exemplars | Completeness >= 0.80 cho tập Hard (H01-H05) | Kiểm tra tỉ lệ trích xuất đủ thông tin các vế trong các câu hỏi phức tạp. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* `run_regression()` phải được tích hợp tự động vào CI/CD Pipeline (GitHub Actions / GitLab CI) và được kích hoạt ở các thời điểm:
> 1. Mỗi khi có Pull Request (PR) thay đổi System Prompt, RAG code, retriever (BM25/Embedding), hoặc chunking strategy.
> 2. Trước khi merge code vào nhánh `main` / `production`.
> 3. Định kỳ hàng tuần để phát hiện rủi ro model drift nếu dùng LLM API bên ngoài.

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> *Câu trả lời:* Threshold 0.05 (giảm tối đa 5% điểm) là **PHÙ HỢP** đối với tổng thể hệ thống hỗ trợ sinh viên vì nó cho phép dung sai vừa phải trước những biến động nhỏ ngẫu nhiên (stochastic variations) của LLM. Tuy nhiên, đối với chỉ số **Faithfulness** (tránh hallucination sai lệch chính sách tiền bạc/học tập) hoặc các quy định bảo mật, cần đặt threshold nghiêm ngặt hơn (0.02) để ngăn chặn rủi ro sai sót pháp lý/tài chính.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment (Zero/Low Tolerance):**
>   - **Faithfulness drop > 0.02** hoặc xuất hiện lỗi **Hallucination**: Chặn ngay lập tức vì cung cấp thông tin sai cho sinh viên có thể dẫn đến khiếu nại.
>   - **Context Recall drop > 0.05**: Chặn deployment nếu khả năng tìm kiếm bối cảnh nguồn bị suy giảm.
>   - **Tỉ lệ Pass Rate giảm > 5%**.
> - **Alert Only (Non-blocking):**
>   - **Answer Relevance / Completeness drop nhỏ (< 0.05)**: Phát cảnh báo lên kênh Slack/Teams của dev team để rà soát prompt mà không cần block ngay deployment khẩn cấp.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [ Unit Testing & Synthetic Eval ] → [ Offline Golden Dataset Benchmark ] → [ Regression Check against Baseline ] → Deploy
```

> *Giải thích:*
> 1. **Unit Testing & Synthetic Eval:** Kiểm tra cú pháp, logic hàm và test nhanh trên tập prompt nhỏ.
> 2. **Offline Golden Dataset Benchmark:** Chạy toàn bộ 20 QA Golden Dataset qua `BenchmarkRunner` để tính 5 chỉ số RAG Metrics.
> 3. **Regression Check against Baseline:** Kích hoạt `run_regression()` so sánh điểm mới với điểm baseline đã lưu trong production; nếu có chỉ số block bị vi phạm thì hủy deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Cập nhật System Prompt hỗ trợ định dạng câu trả lời cấu trúc & loại bỏ meta-talk | Relevance & Faithfulness | Tăng Pass Rate từ 35% lên >= 75% |
| 2 | Bổ sung Guardrail phân loại ý định (Intent Classification) cho câu hỏi out-of-scope | Answer Relevance (Adversarial) | Loại bỏ danh sách dịch vụ thừa, câu trả lời súc tích |
| 3 | Tích hợp Reranker (Cross-Encoder) cho khâu Retrieval | Context Precision & Recall | Đạt Context Precision >= 0.98 trên các truy vấn phức tạp |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Case Bẫy Hoàn Học Phí (False Premise):** *"Vì sinh viên xin rút học phần do ốm sau ngày census luôn được hoàn 100% tiền mặt, quy trình nhận lại tiền tại văn phòng Tài chính là gì?"* (Kiểm tra khả năng từ chối tiền đề sai về chính sách hoàn tiền mặt).
> 2. **Case Yêu Cầu Ngoài Phạm Vi (Out-of-scope Legal):** *"Bạn có thể soạn thảo hợp đồng thuê nhà ngoài ký túc xá cho tôi không?"* (Kiểm tra khả năng từ chối ngắn gọn yêu cầu pháp lý).
> 3. **Case Ràng Buộc Phức Tạp (Hard Multi-Condition):** *"Sinh viên bị kỷ luật cảnh cáo có được đăng ký quá 18 tín chỉ trong kỳ Hè nếu điểm GPA là 3.50 không?"* (Kiểm tra xử lý đồng thời quy định về hold kỷ luật, giới hạn tín chỉ kỳ Hè và điều kiện GPA).

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:* Điều bất ngờ nhất là **Retrieval bằng BM25 hoạt động cực kỳ xuất sắc** (Context Precision đạt 0.943, Context Recall đạt 0.878), vượt xa kỳ vọng đối với một thuật toán tìm kiếm từ khóa truyền thống. Ngược lại, điểm yếu lớn nhất lại nằm ở khâu Generation của LLM mạnh như `llama-3.3-70b-versatile`: model dễ sinh ra văn bản rườm rà (meta-talk) hoặc quá vắn tắt không khớp n-gram với expected answer, làm giảm điểm số đánh giá dù kiến thức trả lời hoàn toàn đúng.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> - **Giới hạn của Word-Overlap Heuristics:**
>   - **Quá nhạy cảm với cách diễn đạt:** Phạt nặng câu trả lời đúng nếu LLM dùng từ đồng nghĩa (ví dụ: "refunded" vs "reversed", "no grade" vs "records a drop").
>   - **Không đo lường được ngữ nghĩa thật sự:** Không phân biệt được câu trả lời đúng ngắn gọn với câu trả lời sai lệch nếu số lượng từ trùng lặp như nhau.
>   - **Bị nhiễu bởi meta-talk:** Thêm các từ dẫn nhập bối cảnh làm giảm điểm Relevance dù thông tin chính vẫn đúng.
> - **Đề xuất nâng cấp cho Production:**
>   1. **Thay thế bằng Embedding Semantic Similarity:** Dùng Cosine Similarity của Dense Embeddings (như OpenAI `text-embedding-3-large` hoặc BGE) để đo độ tương đồng ngữ nghĩa giữa Actual Answer và Expected Answer.
>   2. **Sử dụng LLM-as-a-Judge (Chấm điểm bằng AI Judge):** Dùng prompt chấm điểm dựa trên Rubric 1-5 điểm cho 3 tiêu chí Correctness, Groundedness, và Conciseness thay vì đếm n-gram.
>   3. **Bổ sung Metric An toàn & Bảo mật (Safety & PII Guardrails):** Kiểm tra rò rỉ thông tin cá nhân (FERPA) và khả năng kháng Prompt Injection.
