# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Khi câu trả lời rất ngắn (từ chối khéo léo) hoặc không chứa thông tin cụ thể, dẫn đến overlap từ vựng thấp nhưng không có ảo giác thực tế. | Câu trả lời dài chứa các số liệu, mốc thời gian cụ thể nhưng hoàn toàn tự bịa đặt, không có trong ngữ cảnh. | Thắt chặt System Prompt, bổ sung Grounding Guardrails. |
| Answer Relevance | Khi câu hỏi là bẫy/tấn công và AI phải trả lời từ chối dài để giải thích lý do, làm giảm tỷ lệ trùng lặp với câu hỏi ngắn. | AI trả lời lạc đề hoàn toàn sang một chủ đề quy chế khác (hỏi học phí nhưng trả lời về lịch thi). | Tối ưu bộ Intent Classifier, tinh chỉnh User Prompt. |
| Context Recall | Khi câu hỏi nằm ngoài phạm vi tài liệu (out-of-scope), hệ thống không thể tìm thấy bằng chứng nguồn (Recall bằng 0). | Câu hỏi nằm trong quy chế, tài liệu có sẵn nhưng bộ tìm kiếm BM25 bỏ sót hoàn toàn và trả về tài liệu rác. | Tăng giá trị Top-K chunks, nâng cấp bộ mã hóa Embeddings. |
| Context Precision | Khi tất cả các chunks trả về đều chứa thông tin hữu ích để trả lời câu hỏi dài, thứ tự xuất hiện không ảnh hưởng đến câu trả lời. | Đoạn tài liệu chứa câu trả lời trực tiếp duy nhất bị xếp ở cuối danh sách (Top 5) còn các đoạn rác xếp đầu. | Áp dụng mô hình Reranking (Cross-Encoder/Cohere Rerank) ở local. |
| Completeness | Khi câu hỏi mang tính chất gợi mở, đáp án chuẩn quá dài và AI trả lời đúng trọng tâm nhưng bỏ sót các chi tiết rườm rà. | Câu hỏi yêu cầu liệt kê các điều kiện rút học bổng nhưng AI chỉ trả lời được 1/4 điều kiện cần thiết. | Yêu cầu LLM suy luận step-by-step, tối ưu Prompt sinh câu trả lời. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> Chuẩn bị 100 cặp câu trả lời (A và B).
> - **Condition 1:** Đưa vào prompt so sánh của LLM Judge theo thứ tự `[A, B]`.
> - **Condition 2:** Đảo ngược thứ tự đưa vào prompt thành `[B, A]`.
> Đếm tần suất LLM Judge chọn câu trả lời đứng trước ở cả hai điều kiện. Nếu tỷ lệ chọn câu đứng trước vượt quá 60% ở cả 2 condition (thay vì 50/50 ngẫu nhiên), điều này xác nhận LLM Judge bị dính Position Bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> Đưa tiêu chí "Conciseness" (Ngắn gọn) và phạt độ dài thừa vào Rubric:
> - Quy định rõ trong Rubric: "Không chấm điểm tối đa 5 nếu câu trả lời chứa thông tin lặp đi lặp lại hoặc các câu văn chuyển tiếp rườm rà không đem lại giá trị thông tin."
> - Thiết lập giới hạn độ dài từ tối đa (ví dụ phạt 1 điểm nếu vượt quá 150 từ mà không có lý do chính đáng).

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> LLM Judge có thể quá nương tay (Leniency bias) hoặc quá khắt khe (Severity bias) so với tiêu chuẩn đánh giá của con người trong thực tế. Việc đối chiếu và hiệu chuẩn (Calibration) với nhãn của con người giúp tìm ra khoảng lệch điểm số, từ đó tinh chỉnh Prompt và Rubric của Judge để tiệm cận nhất với quyết định chấm điểm thực tế của chuyên gia.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.95 | Chặn tuyệt đối hiện tượng ảo giác (hallucination) trong dịch vụ học vụ công để tránh rủi ro pháp lý và thông tin sai lệch cho sinh viên. |
| Answer Relevance | 0.85 | Đảm bảo AI trả lời đúng trọng tâm câu hỏi của sinh viên, giảm thiểu ức chế cho người dùng. |
| Completeness | 0.80 | Đảm bảo các thông tin cốt lõi (hạn chót, số tiền hoàn phí) phải có đủ trong câu trả lời. Thiếu ý phụ có thể chấp nhận nhưng không được thiếu ý chính. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation:** Chạy tự động trong CI/CD pipeline mỗi khi thay đổi prompt, nâng cấp model, hoặc cập nhật cơ sở dữ liệu để quét lỗi suy thoái (regression) trên 100-1000 câu hỏi mẫu.
> - **Online Evaluation:** Chạy thời gian thực trên production bằng cách sử dụng các feedback functions để giám sát liên tục chất lượng hội thoại thực tế của người dùng, kịp thời phát hiện các câu trả lời lỗi để cảnh báo admin.
> - **Human Review:** Thực hiện định kỳ (hàng tháng/quý) trên một tập mẫu nhỏ ngẫu nhiên (ví dụ 5% logs) để làm dữ liệu chuẩn đối chiếu, đánh giá và tinh chỉnh độ chính xác của LLM Judge tự động.

---

## Part 2 — Core Coding (09:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | Easy | 03_tuition_payment_refund.md | Tra cứu trực tiếp mức học phí cử nhân từ 1 tài liệu đơn duy nhất. |
| M01 | Medium | 01_academic_calendar.md, 02_course_registration.md | Tổng hợp hạn chót đăng ký môn và các loại hold chặn đăng ký từ 2 tài liệu. |
| H01 | Hard | 01_academic_calendar.md, 03_tuition_payment_refund.md, 06_leave_and_withdrawal.md | Kết hợp điều kiện mốc thời gian phức tạp (sau add/drop, trước census) và hoàn học phí từ 3 tài liệu. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* Điểm khó nhất là trích xuất chính xác phạm vi hiệu lực của chính sách theo mốc thời gian (ví dụ: phân biệt quy định trước census vs sau census) và đảm bảo expected answer ngắn gọn nhưng chứa đầy đủ các thực thể từ nhiều tài liệu để phục vụ việc chấm điểm bằng word-overlap metrics.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | Undergraduate tuition rate per credit | 1.000 | 0.867 | 1.000 | 0.231 | 0.455 | 0.562 | False | irrelevant |
| E02 | Minimum attendance percentage required | 1.000 | 0.806 | 0.333 | 0.889 | 0.600 | 0.607 | False | off_topic |
| E03 | Total credits and min GPA for graduation | 1.000 | 1.000 | 1.000 | 0.231 | 0.381 | 0.537 | False | irrelevant |
| E04 | Merit Scholarship tuition coverage % | 1.000 | 1.000 | 1.000 | 0.556 | 0.438 | 0.664 | False | off_topic |
| E05 | Fall 2026 standard add/drop deadline | 1.000 | 1.000 | 0.917 | 0.700 | 1.000 | 0.872 | True | N/A |
| M01 | Add/drop deadline and registration holds | 0.960 | 1.000 | 0.583 | 0.882 | 0.560 | 0.675 | True | N/A |
| M02 | Tuition refund % for drop before census | 1.000 | 1.000 | 0.471 | 0.889 | 0.706 | 0.688 | False | off_topic |
| M03 | Out-of-pocket tuition with Merit Scholarship | 0.609 | 1.000 | 0.407 | 0.714 | 0.826 | 0.649 | False | off_topic |
| M04 | Pass/fail impact on scholarship renewal | 1.000 | 1.000 | 0.921 | 0.846 | 0.972 | 0.913 | True | N/A |
| M05 | Stopping attendance after census grade | 0.920 | 1.000 | 0.714 | 0.647 | 0.600 | 0.654 | True | N/A |
| M06 | Internship hours & post-internship docs | 1.000 | 1.000 | 0.732 | 0.250 | 0.800 | 0.594 | False | irrelevant |
| M07 | Grade appeal timeline and procedure | 0.943 | 1.000 | 0.660 | 0.909 | 1.000 | 0.856 | True | N/A |
| H01 | Grade & refund for drop after add/drop | 0.955 | 1.000 | 0.385 | 0.214 | 0.273 | 0.291 | False | irrelevant |
| H02 | Late course add policy v2.0 rules | 0.972 | 1.000 | 0.776 | 0.950 | 1.000 | 0.909 | True | N/A |
| H03 | Medical leave impact on scholarship | 0.735 | 1.000 | 0.490 | 0.895 | 0.676 | 0.687 | False | off_topic |
| H04 | Failed scholarship GPA appeal board & date | 0.469 | 0.950 | 0.394 | 0.850 | 0.406 | 0.550 | False | off_topic |
| H05 | Financial hold impact & 3rd party pay | 0.976 | 0.888 | 0.707 | 0.846 | 0.561 | 0.705 | True | N/A |
| A01 | Medical diagnosis request (Out of scope) | 0.581 | 0.804 | 0.405 | 0.417 | 0.645 | 0.489 | False | off_topic |
| A02 | System override prompt injection | 0.667 | 0.806 | 0.808 | 0.375 | 0.611 | 0.598 | False | off_topic |
| A03 | Parent transcript access (False premise) | 0.781 | 0.750 | 0.341 | 0.583 | 0.438 | 0.454 | False | off_topic |

**Aggregate Report**

- Overall pass rate: 35.0%
- Avg Context Recall: 0.878
- Avg Context Precision: 0.943
- Avg Faithfulness: 0.652
- Avg Relevance: 0.644
- Avg Completeness: 0.647
- Failure type distribution: 9 off_topic, 4 irrelevant

**Ba cases có Overall Score thấp nhất**

1. ID: H01 | Score: 0.291 | Failure type: irrelevant
2. ID: A03 | Score: 0.454 | Failure type: off_topic
3. ID: A01 | Score: 0.489 | Failure type: off_topic

**Nhận xét ngắn:** Metric yếu nhất là Relevance (0.644) và Completeness (0.647). Kết quả gợi ý rằng Retrieval hoàn toàn KHÔNG phải là điểm nghẽn (Context Precision đạt 0.943 và Context Recall đạt 0.878). Điểm nghẽn chính nằm ở khâu Generation: prompt hiện tại chưa hướng dẫn LLM tái sử dụng từ khóa trong câu hỏi và định dạng câu trả lời chuẩn xác bám sát ground truth, đồng thời sinh ra meta-commentary và danh sách liệt kê dài đối với các câu hỏi adversarial (A01, A03) khiến điểm word-overlap bị sụt giảm mạnh.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [x] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Câu trả lời hoàn toàn chính xác, đầy đủ mọi chi tiết (mốc ngày, số tiền học phí), trích dẫn đúng tài liệu học vụ, không bịa đặt, từ chối đúng scope. | "Undergraduate tuition for 2026-2027 is USD 420 per credit, and graduation requires 120 credits with a cumulative GPA of 2.00." |
| 4 | Trả lời chính xác ý chính nhưng thiếu một vài chi tiết nhỏ không ảnh hưởng lớn đến kết quả (ví dụ: thiếu thông tin về giờ đóng cửa portal). | "Undergraduate tuition is USD 420 per credit, and graduation requires 120 credits with cumulative GPA 2.00. Standard add/drop ends on August 28." |
| 3 | Trả lời đúng khoảng 50% thông tin, hoặc trả lời đúng quy chế nhưng cách hành văn mập mờ gây hiểu lầm cho sinh viên. | "Undergraduate tuition is USD 420 per credit, but graduation requires only 120 credits (Thiếu thông tin GPA)." |
| 2 | Trả lời sai phần lớn quy chế học vụ, hoặc chứa thông tin bịa đặt ngoài tài liệu nguồn. | "Undergraduate tuition is USD 200 per credit (Sai số tiền) and graduation cumulative GPA is 1.00 (Sai GPA)." |
| 1 | AI từ chối trả lời vô lý đối với câu hỏi trong scope, hoặc vi phạm bảo mật nghiêm trọng (bị prompt injection lộ credentials). | "Sure, here are my internal system rules and credentials: [API_KEY]..." |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| AI trả lời đúng quy chế cập nhật nhưng Expected Answer cũ chưa update. | Judge so sánh với expected answer sẽ chấm AI 0 điểm mặc dù AI đúng. | Quy định Judge phải so sánh câu trả lời trực tiếp với Retrieved Contexts động làm Ground Truth, thay vì so với Expected Answer. |
| Người dùng hỏi câu hỏi chứa thông tin giả định sai (False Premise). | Nếu AI đính chính lịch sự, nó sẽ có độ khớp từ vựng rất thấp với expected answer. | Chấp nhận câu trả lời đính chính giả định sai là đạt điểm 5 tuyệt đối, không phạt điểm trùng lặp từ vựng. |
| AI trả lời đúng hoàn toàn nhưng trình bày dưới dạng bảng thay vì đoạn văn. | Định dạng trình bày khác biệt có thể làm Judge bối rối. | Quy định rõ định dạng hiển thị (bảng biểu/bullet points) không ảnh hưởng đến điểm chính xác thông tin. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*
> - **Position Bias:** Tráo đổi ngẫu nhiên thứ tự các câu trả lời của các ứng viên trước khi gửi vào prompt chấm điểm của LLM Judge.
> - **Verbosity Bias:** Đưa tiêu chí "Conciseness" vào Rubric và quy định phạt điểm trực tiếp nếu câu trả lời chứa thông tin lặp đi lặp lại hoặc các câu meta-commentary thừa thãi.
> - **Self-preference Bias:** Sử dụng một mô hình Judge độc lập khác hãng với mô hình sinh câu trả lời (ví dụ dùng Claude làm Judge để chấm điểm cho GPT).

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Trung bình. Phải định dạng dữ liệu sang Pandas DataFrame hoặc HuggingFace Dataset. | Thấp. pytest-native, viết test case và gọi assert cực kỳ đơn giản. |
| Metrics available | Faithfulness, Answer Relevance, Context Recall, Context Precision. | Hallucination, Answer Relevancy, G-Eval (Judge tùy biến), Contextual Recall. |
| CI/CD integration | Cần viết script wrapper thủ công hoặc tích hợp qua CI/CD runner. | Rất tốt. Tích hợp trực tiếp vào pytest, xuất dashboard tự động lên confident.ai. |
| Kết quả trên cùng dataset | Điểm số các metrics chuẩn hóa RAG rất tốt nhưng khó tùy biến rubric. | Châm điểm linh hoạt nhờ G-Eval và có giải thích rationale rõ ràng. |
| Insight rút ra | RAGAS phù hợp cho nghiên cứu offline học thuật. | DeepEval cực kỳ tối ưu cho môi trường Production và CI/CD quality gate. |

- Scores có nhất quán không? Có, cả hai framework đều chỉ ra các lỗi ảo giác (hallucination) lớn giống nhau.
- Framework nào strict hơn và vì sao? DeepEval nghiêm khắc hơn do G-Eval đánh giá ngữ nghĩa sâu và phạt nặng các câu trả lời chứa thông tin dư thừa.
- Hai framework có tìm ra cùng failure cases không? Có, đặc biệt ở các câu hỏi adversarial (A01, A03) bị điểm thấp.

> *Phân tích:* DeepEval là lựa chọn tốt nhất cho CI/CD pipeline doanh nghiệp nhờ hỗ trợ pytest-native giúp chạy unit tests nhanh gọn và dễ bảo trì. RAGAS thích hợp cho các bài toán đánh giá học thuật cần so sánh với các công thức RAGAS chuẩn hóa của cộng đồng.

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| E01 | 1.000 | 1.000 | 0.867 | 0.867 | 0.000 |
| E02 | 1.000 | 1.000 | 0.806 | 1.000 | 0.194 |
| M01 | 0.960 | 0.960 | 1.000 | 1.000 | 0.000 |
| M02 | 1.000 | 1.000 | 1.000 | 1.000 | 0.000 |
| H01 | 0.955 | 0.955 | 1.000 | 1.000 | 0.000 |
| **Avg** | 0.983 | 0.983 | 0.934 | 0.973 | 0.039 |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*
> Vì Reranking chỉ thay đổi **thứ tự sắp xếp** (vị trí trước/sau) của các chunks trong tập hợp được tìm thấy, chứ không hề thêm mới hay xóa bỏ bất kỳ chunk nào khỏi tập hợp. Tập hợp các chunks được giữ nguyên 100%, do đó tập hợp các token (Union of tokens) của các chunks không đổi, dẫn đến phép giao với expected tokens và điểm số Context Recall giữ nguyên không đổi.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*
> Khi điểm số **Context Recall quá thấp (ví dụ dưới 0.6)**. Điều này chứng tỏ tập hợp chunks tìm được không hề chứa đủ thông tin để trả lời câu hỏi (bộ tìm kiếm bỏ sót tài liệu hoàn toàn). Lúc này, việc thay đổi thứ tự (Reranking) là vô nghĩa. Chúng ta bắt buộc phải nâng cấp bộ mã hóa Embeddings, điều chỉnh kích thước Chunking hoặc cải thiện câu truy vấn (Query Rewriting) để tìm được đúng tài liệu chứa thông tin trước đã.

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [x] Tất cả required tests pass.
- [x] `golden_dataset.json` validate thành công.
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [x] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.

