# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 9:15–12:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 9:15–9:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (9:30–9:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Low recall của retriever khiến context không đủ để answer đầy đủ | Model bịa thông tin (hallucination) khi context có đủ nhưng model vẫn thêm thông tin sai | Thêm guardrails, cải thiện retrieval |
| Answer Relevance | Question ambiguous hoặc multi-intent không rõ ràng | Answer hoàn toàn không liên quan đến câu hỏi | Cải thiện prompt, thêm intent classification |
| Context Recall | Acceptable khi relevant chunks bị trùng lặp trong retrieval | Retriever miss critical evidence mà corpus có | Cải thiện chunking, tăng overlap, thử retriever khác |
| Context Precision | Acceptable khi nhiều relevant chunks có độ quan trọng tương đương | Relevant chunks bị buried dưới noise ở vị trí cao | Implement reranking, cải thiện retrieval quality |
| Completeness | Acceptable khi expected answer quá dài hoặc có nhiều optional details | Model bỏ sót key requirements từ policy | Tăng context window, cải thiện generation |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> 
> **Experiment Design:**
> - **Condition A (Baseline):** Hiển thị answer A trước, answer B sau. Ghi điểm judge cho cả hai.
> - **Condition B (Reversed):** Hiển thị answer B trước, answer A sau. Ghi điểm judge cho cả hai.
> - **Metric:** So sánh điểm của cùng một answer khi ở vị trí first vs second.
> - **Expected:** Nếu có position bias, answer ở vị trí first sẽ được điểm cao hơn trong cả hai conditions.
> - **Statistical Test:** Paired t-test với α = 0.05

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 
> 1. **Normalize by length:** Đánh giá dựa trên content density, không phải raw length
> 2. **Penalize redundancy:** Explicit rubric rule: "Answer bị trừ điểm nếu chứa repeated information không thêm giá trị"
> 3. **Fixed-length baseline:** So sánh với expected answer length thay vì dùng độ dài tuyệt đối
> 4. **Length-agnostic scoring:** Score dựa trên information coverage (completeness) và correctness, không reward length
> 5. **Edge cases:** Ngưỡng min/max length - answer quá ngắn (<threshold) hoặc quá dài (>threshold) đều bị trừ

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 
> 1. **Establish ground truth:** Human labels cung cấp objective baseline để so sánh judge performance
> 2. **Detect systematic biases:** Calibration giúp phát hiện judge consistently over/under-score so với human consensus
> 3. **Confidence calibration:** Điều chỉnh confidence levels của judge để align với actual agreement rates
> 4. **Cross-model comparison:** Không thể so sánh judges giữa các models nếu không có unified calibration standard
> 5. **Quality assurance:**确保 judge không introduce new biases khi evaluate

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.7 | Threshold cao vì hallucination có thể gây hại cho customer (sai thông tin về warranty, returns, etc.) |
| Answer Relevance | 0.6 | Chấp nhận lower threshold vì có thể do question ambiguity |
| Completeness | 0.6 | Threshold vừa phải - đảm bảo answer cover key requirements nhưng không quá strict |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> 
> **Offline Evaluation:**
> - Sau mỗi code change, prompt modification, hoặc model upgrade
> - Trước khi deploy feature mới
> - Regression testing hàng ngày
> - Fast feedback loop với golden dataset cố định
> 
> **Online Evaluation:**
> - A/B testing với live traffic
> - Real-time monitoring của production metrics
> - Phát hiện concept drift hoặc distribution shift
> - Khi cần sample size lớn để detect subtle improvements
> 
> **Human Review:**
> - Sensitive cases (refunds, account security, warranty claims)
> - When automated metrics không align với user satisfaction
> - Edge cases và adversarial inputs
> - Periodic quality audit của automated pipeline

---

## Part 2 — Core Coding (9:45–10:40)

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
được điền trực tiế trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
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
| H01 | Hard | 09_escalation_and_policy_updates.md | Requires understanding of policy versioning - order date determines which policy applies, not delivery date |
| A01 | Adversarial (out_of_scope) | 00_system_scope.md | Tests refusal behavior - legal matters are explicitly out of scope per system scope document |
| H02 | Hard | 06_warranty_policy.md, 07_repair_and_technical_support.md | Requires multi-document reasoning - warranty vs return policy interaction after window closes |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*
> 
> Điểm khó nhất là đảm bảo expected answer ngắn gọn nhưng đầy đủ thông tin cần thiết (dates, amounts, conditions, exceptions) mà không có extra claims không có trong corpus. Hard cases đặc biệt khó vì cần kết hợp nhiều documents và context rất dễ bị split không chính xác. Adversarial cases cần viết expected answer đúng với policy behavior (refuse correctly) nhưng không quá robotic.

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

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | NovaBook 14 ports | 1.000 | 1.000 | 0.889 | 0.500 | 1.000 | 0.796 | ✓ | - |
| E02 | Standard shipping time | 1.000 | 1.000 | 0.909 | 0.500 | 0.909 | 0.773 | ✓ | - |
| E03 | PulsePhone X warranty | 0.875 | 1.000 | 0.800 | 0.600 | 0.500 | 0.633 | ✓ | - |
| E04 | Unopened return window | 1.000 | 1.000 | 0.941 | 0.800 | 0.917 | 0.886 | ✓ | - |
| E05 | OrbitPlus cost | 1.000 | 0.950 | 0.667 | 0.333 | 0.667 | 0.556 | ✗ | off_topic |
| M01 | Order cancellation | 1.000 | 0.888 | 0.621 | 0.556 | 0.692 | 0.623 | ✓ | - |
| M02 | OrbitPlus refund | 1.000 | 1.000 | 0.606 | 0.846 | 0.714 | 0.722 | ✓ | - |
| M03 | Restocking fee | 0.933 | 1.000 | 0.556 | 0.750 | 0.667 | 0.657 | ✓ | - |
| M04 | Warranty repair req | 1.000 | 1.000 | 0.553 | 0.600 | 0.840 | 0.664 | ✓ | - |
| M05 | Report damage | 1.000 | 0.888 | 0.840 | 0.889 | 0.955 | 0.894 | ✓ | - |
| M06 | Account compromise | 1.000 | 0.806 | 0.511 | 0.750 | 0.935 | 0.732 | ✓ | - |
| M07 | OrbitPay failure | 1.000 | 1.000 | 0.538 | 0.714 | 0.667 | 0.640 | ✓ | - |
| H01 | Policy version apply | 0.950 | 1.000 | 0.773 | 0.571 | 0.750 | 0.698 | ✓ | - |
| H02 | Defect after return | 0.419 | 1.000 | 0.100 | 0.643 | 0.290 | 0.344 | ✗ | hallucination |
| H03 | Discount stacking | 0.917 | 0.888 | 0.778 | 0.900 | 0.625 | 0.768 | ✓ | - |
| H04 | Lost proof of purchase | 0.846 | 0.917 | 0.500 | 0.917 | 0.500 | 0.639 | ✓ | - |
| H05 | Bundle gift return | 0.958 | 0.950 | 0.550 | 0.818 | 0.708 | 0.692 | ✓ | - |
| A01 | Legal letter request | 0.360 | 1.000 | 0.071 | 0.333 | 0.320 | 0.242 | ✗ | hallucination |
| A02 | Credit card request | 0.735 | 1.000 | 0.364 | 0.333 | 0.147 | 0.281 | ✗ | incomplete |
| A03 | 2-year warranty claim | 0.457 | 0.917 | 0.079 | 0.750 | 0.371 | 0.400 | ✗ | hallucination |

**Aggregate Report**

- Overall pass rate: **75%**
- Avg Context Recall: **0.873**
- Avg Context Precision: **0.960**
- Avg Faithfulness: **0.582**
- Avg Relevance: **0.655**
- Avg Completeness: **0.659**
- Failure type distribution: hallucination=3, incomplete=1, off_topic=1

**Ba cases có Overall Score thấp nhất**

1. ID: A01 | Score: 0.242 | Failure type: hallucination
2. ID: A02 | Score: 0.281 | Failure type: incomplete
3. ID: H02 | Score: 0.344 | Failure type: hallucination

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*
> 
> **Faithfulness là metric yếu nhất (0.582)** - đây là vấn đề **generation**, không phải retrieval. Retrieval metrics cao (Recall=0.873, Precision=0.960) cho thấy retriever hoạt động tốt. Tuy nhiên, model vẫn tạo ra hallucination - thêm thông tin không có trong context (A01, A02, A03) hoặc không paraphrase đúng context (H02).
> 
> Adversarial cases (A01, A02, A03) cho thấy model khó handle out-of-scope và prompt injection - điểm rất thấp dù retrieval có thể đã lấy được scope policy.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 4 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Evidence/citation
- [ ] Actionability
- [x] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | **Correct + Complete + Well-cited:** Answer đúng 100%, cover tất cả key requirements, cite đúng policy sections. Không có extra claims không có trong corpus. | "According to Return Policy v2.0, you have 30 days to return an unopened device. The return window starts from confirmed delivery date." |
| 4 | **Mostly correct + Minor gaps:** Answer đúng nhưng thiếu 1-2 details không critical (ví dụ: thiếu exact time period, conditions). Có citations nhưng không đầy đủ. | "You can return the device within 30 days. The policy covers unopened devices." (thiếu "from confirmed delivery") |
| 3 | **Partially correct + Some errors:** Answer đúng direction nhưng có 1-2 factual errors hoặc missing key conditions. References đúng documents nhưng không trích dẫn cụ thể. | "You have 21 days to return" (sai date cho v2.0) |
| 2 | **Significant errors or Missing info:** Answer có major errors hoặc bỏ sót critical policy information (warranty terms, refund conditions). Không có citations. | "You can return anytime within a month" (quá vague, không có policy basis) |
| 1 | **Wrong or Dangerous:** Answer hoàn toàn sai fact, hướng dẫn sai policy, hoặc potentially harmful. Fabricated policy information. | "You can return within 90 days" (không có trong any policy) |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Policy version ambiguity | Customer hỏi nhưng không cung cấp order date, answer có thể đúng với 1 version nhưng sai với version khác | Score 3 nếu identify both possibilities và request order date; Score 2 nếu pick 1 version không hỏi; Score 1 nếu guess wrong version |
| Boundary dates (exact 2 years) | "Exactly 2 years ago" có thể mean warranty vừa expire hoặc chưa expire tùy delivery date | Score 4 nếu mention uncertainty và suggest checking delivery date; Score 2 nếu give definitive answer mà không qualify |
| Refusal on borderline cases | Case có thể partially in-scope nhưng model refuse hoặc over-refuse | Score based on whether refusal đúng policy (A01) vs overly cautious (M07 borderline) |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*
> 
> **Position Bias Control:**
> - Randomize answer order trong prompt
> - Đánh giá từng answer độc lập, không so sánh trực tiếp
> - Swap positions giữa runs và average scores
> 
> **Verbosity Bias Control:**
> - Normalize scores by content density thay vì raw length
> - Penalize redundant information explicitly
> - Use length-agnostic rubric - score based on information coverage, not word count
> 
> **Self-Preference Bias Control:**
> - Use different judge model (e.g., GPT-4 for judging, GPT-4o-mini for generation)
> - Include diverse answer styles trong calibration set
> - Calibrate against human labels từ multiple annotators

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Medium - requires OpenAI API, separate installation | Low - single pip install, Python-native |
| Metrics available | 5+ (faithfulness, answer_relevancy, context_recall, context_precision, response_matching) | 8+ (faithfulness, answer_relevancy, context_precision, hallucination, toxicity, etc.) |
| CI/CD integration | Requires custom wrapper, not native | Native pytest integration with `@pytest.mark.asyncio` |
| Kết quả trên cùng dataset | Word-overlap based, may differ from LLM-based | LLM-evaluated, potentially higher correlation with human judgment |
| Insight rút ra | Fast, deterministic, good for regression testing | More nuanced, captures semantic similarity better |

- **Scores có nhất quán không?** Không, vì RAGAS dùng word-overlap heuristic trong khi DeepEval dùng LLM-based evaluation. Cùng dataset có thể cho scores khác nhau 10-20%.
- **Framework nào strict hơn và vì sao?** DeepEval strict hơn vì LLM-based evaluation capture semantic nuance tốt hơn word overlap. Word-overlap có thể inflated nếu có nhiều shared stopwords.
- **Hai framework có tìm ra cùng failure cases không?** Thường tìm ra cùng top failures nhưng ranking có thể khác. DeepEval thường identify thêm semantic issues mà word-overlap miss.

> *Phân tích:*
> 
> RAGAS phù hợp cho rapid iteration và regression testing vì deterministic và fast. DeepEval phù hợp cho final quality gate vì nuanced evaluation. Recommend: dùng cả hai - RAGAS cho CI pipeline, DeepEval cho human-in-the-loop review.

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
| E01 | 1.000 | 1.000 | 1.000 | 1.000 | 0.000 |
| M01 | 1.000 | 1.000 | 0.888 | 0.933 | +0.045 |
| H01 | 0.950 | 0.950 | 1.000 | 1.000 | 0.000 |
| H03 | 0.917 | 0.917 | 0.888 | 0.917 | +0.029 |
| A02 | 0.735 | 0.735 | 1.000 | 1.000 | 0.000 |
| **Avg** | 0.920 | 0.920 | 0.955 | 0.970 | **+0.015** |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*
> 
> Recall đo coverage dựa trên **union** của tất cả chunks. Khi rerank, ta chỉ thay đổi thứ tự, không thêm hay bớt chunks. Vì vậy union của tokens vẫn giữ nguyên, và recall = |expected ∩ union| / |expected| không thay đổi.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*
> 
> 1. **Recall < 0.7:** Relevant chunks không được retrieve → cần improve retriever hoặc index
> 2. **Precision không improve sau rerank:** Noise chunks chiếm majority → cần better chunking strategy hoặc hybrid search
> 3. **Query-chunk semantic gap:** Query vocabulary khác chunk vocabulary → cần query expansion hoặc dense retrieval
> 4. **Chunk size không phù hợp:** Quá nhỏ → miss context; Quá lớn → dilute relevance → cần tune chunk size

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
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
