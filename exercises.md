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
| Faithfulness | Câu trả lời sáng tạo, brainstorming hoặc hội thoại không yêu cầu mọi ý phải xuất hiện nguyên văn trong context. | Bài toán factual/high-stakes nhưng answer chứa nhiều claim không được gold context hỗ trợ; đặc biệt nghiêm trọng khi retrieval scores vẫn tốt. | Kiểm tra các claim không grounded; siết prompt yêu cầu chỉ dùng context, thêm citation/grounding guardrail và cơ chế từ chối khi thiếu evidence. |
| Answer Relevance | Câu hỏi mở cần thêm giải thích nền hoặc ví dụ hữu ích, khiến overlap từ vựng với question thấp dù answer vẫn đúng mục tiêu. | Answer không giải quyết intent chính, trả lời sai chủ đề hoặc bị routing sang domain khác. | Phân tích intent và các case off-topic; cải thiện prompt, query rewriting/routing và bổ sung test theo từng intent. |
| Context Recall | Câu hỏi có thể trả lời đúng bằng kiến thức ổn định của model, hoặc expected answer chứa chi tiết không cần thiết so với yêu cầu người dùng. | Retriever bỏ sót evidence thiết yếu, đồng thời answer thiếu các ý tương ứng và Completeness thấp. | Kiểm tra corpus/chunking/index; cải thiện query expansion, embedding, top-k và bổ sung tài liệu còn thiếu. |
| Context Precision | Evidence đúng vẫn nằm trong top-k nhưng đứng sau vài chunk nhiễu, trong khi latency và answer quality vẫn đạt yêu cầu. | Phần lớn top-ranked chunks không liên quan, evidence đúng bị đẩy xuống thấp hoặc context window bị noise chiếm hết. | Cải thiện retriever filters và ranking; dùng hybrid search/reranker, điều chỉnh top-k và đo AP@K sau khi rerank. |
| Completeness | Người dùng yêu cầu câu trả lời ngắn/tóm tắt, hoặc expected answer có các chi tiết tùy chọn không cần cho task completion. | Answer bỏ sót các ý bắt buộc, bước an toàn hoặc điều kiện quan trọng; nếu Context Recall cũng thấp thì thường là lỗi retrieval. | So sánh các ý thiếu với retrieved contexts: nếu evidence thiếu thì sửa retrieval; nếu evidence đã có thì sửa generation prompt/checklist. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*

Chuẩn bị nhiều cặp answer A/B có chất lượng tương đương hoặc đã được human
label. Chạy ít nhất hai conditions với cùng question, rubric, judge model và
sampling settings:

1. Condition 1: trình bày A trước B.
2. Condition 2: đảo thứ tự, trình bày B trước A.

Lặp lại trên đủ nhiều cặp và randomize thứ tự giữa các mẫu. So sánh win-rate và
độ lệch score của cùng một answer khi đứng vị trí đầu với khi đứng vị trí sau.
Nếu answer ở vị trí đầu thắng hoặc được tăng điểm một cách có ý nghĩa thống kê,
bất kể nội dung là A hay B, đó là bằng chứng của position bias. Có thể thêm một
condition chấm riêng từng answer để tạo baseline không phụ thuộc thứ tự.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*

Rubric phải nói rõ rằng độ dài tự nó không phải là tiêu chí chất lượng; judge chỉ
chấm mức độ đúng, liên quan, đầy đủ và súc tích. Mỗi criterion cần mô tả bằng
evidence quan sát được, có thang điểm với anchor cụ thể, đồng thời trừ điểm cho
lặp ý, lan man và thông tin không phục vụ câu hỏi. Có thể yêu cầu judge liệt kê
các claim/ý bắt buộc được đáp ứng trước khi cho điểm, nhờ đó một answer ngắn nhưng
đủ ý không bị thua chỉ vì ít từ hơn.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*

Human labels cung cấp chuẩn tham chiếu để biết judge có thật sự đo đúng chất
lượng mong muốn hay chỉ tạo score có vẻ hợp lý. Đối chiếu judge với nhiều người
gán nhãn giúp đo agreement, phát hiện systematic bias, threshold sai và rubric
mơ hồ. Từ các case bất đồng, ta hiệu chỉnh rubric, prompt, score anchors và
deployment threshold. Việc calibration nên được lặp lại khi đổi domain, model
hoặc phân phối dữ liệu; các case high-stakes và bất đồng lớn cần human review.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | >= 0.80 | Claim không grounded tạo rủi ro hallucination, nên đây là gate nghiêm ngặt; dưới 0.80 phải phân tích và dưới 0.60 phải block tuyệt đối. |
| Answer Relevance | >= 0.75 | Cho phép một ít nội dung nền hữu ích nhưng vẫn yêu cầu answer giải quyết trực tiếp intent của người dùng. |
| Completeness | >= 0.75 | Cho phép cách diễn đạt ngắn gọn, nhưng bảo đảm phần lớn ý bắt buộc trong expected answer được bao phủ. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*

- **Offline evaluation:** chạy trên golden dataset trước mỗi release và khi đổi
  prompt, model, retriever hoặc code. Nó lặp lại được, phù hợp regression test và
  là quality gate tự động trong CI/CD. Deployment bị block nếu metric trung bình
  dưới threshold, có regression đáng kể, hoặc case critical/high-stakes thất bại.
- **Online evaluation:** chạy liên tục trên traffic thật sau deployment để phát
  hiện data drift, intent mới, latency/cost bất thường và lỗi chỉ xuất hiện trong
  môi trường production. Dùng sampling, tracing và cảnh báo; không dựa duy nhất
  vào online score để tự động thay đổi hệ thống high-stakes.
- **Human review:** dùng để xây dựng/calibrate golden labels, xử lý case mơ hồ,
  bất đồng giữa evaluators và các quyết định high-stakes. Human feedback cũng
  được đưa trở lại offline dataset cho vòng lặp cải tiến tiếp theo.

Khi **Context Recall thấp và Completeness thấp**, retriever thường không lấy đủ
evidence nên generator không có dữ liệu để bao phủ expected answer. Ngược lại,
nếu Context Recall/Precision tốt nhưng **Faithfulness thấp**, evidence cần thiết
đã được retrieval cung cấp mà answer vẫn tạo claim không grounded; root cause
thường nằm ở generation prompt, model hoặc grounding guardrail.

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
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

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
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
