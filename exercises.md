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

| Metric            | Acceptable Low Score Scenario                                                                          | Critical Low Score Scenario                                                                                        | Action Required                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| Faithfulness      | Model từ chối trả lời hợp lệ do context không đủ thông tin                                             | Model bịa đặt chính sách, thông số kỹ thuật sai sự thật                                                            | Cần thắt chặt system prompt chỉ trả lời thì đủ context và giảm temperature            |
| Answer Relevance  | Người dùng hỏi những câu hỏi xã giao, vô tri và bot trả lời xã giao lại hoặc hỏi thêm thông tin làm rõ | Bot trả lời lạc đề, khách hỏi trạng thái đơn hàng thì lại trả lời chính sách đổi hàng                              | Tối ưu hóa prompt, hướng dẫn bám sát trọng tâm câu hỏi, context của người dùng        |
| Context Recall    | Câu hỏi dạng so sánh hoặc tổng quát mà context chỉ cần một phần dữ liệu để trả lời                     | Context hiếu các điều kiện để áp dụng chính sách bảo hành làm bot trả lời thiếu ý hoặc sai chính sách nghiêm trọng | Tăng top_k, tinh chỉnh embedding, cải thiện chunk strategy                            |
| Context Precision | Cần lấy nhiều ngữ cảnh rộng để bot tổng hợp được các quy định chung                                    | Chunk chứa thông tin mâu thuẫn hoặc rác khiến bot lấy sai dữ liệu ngay từ đầu                                      | Thêm cress-encoder reranker, lọc chunk theo similarity threshold                      |
| Completeness      | Khách hàng hỏi câu ngắn hay yes/no, bot cần trả lời ngắn gọn thay vì dài dòng                          | Khách hàng hỏi thủ tục đổi trả cần 4 bước mà bot trả lời có 1 bước gây hiểu lầm                                    | Cải thiện context recall, bổ xung rubric yêu cầu checklist các bước trước khi trả lời |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**
Condition 1 (original order): đưa prompt theo thứ tự và yêu cầu judge chấm điểm cả 2 và chọn ra câu tốt hơn. Condition 2(swap order) đảo ngược vị trí answer với cùng prompt hay system instruction. Nếu judge luôn chấm điểm câu trước cao hơn câu sau thì dễ là model bị position bias. Giải pháp là chấm điểm cả 2 answer rồi lấy trung bình.
**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

Đặt tiêu chí phạt rõ ràng nếu câu trả lời thêm thông tin thừa hay lặp lại không liên quan. Định nghĩa complete dựa trên số lượng key points thực sự đáp ứng thay vì độ dài hay hoa mỹ.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

Căn chỉnh độ lệch: LLM judge có thể có tiêu chuẩn khắt khe hoặc lỏng lẻo hơn thực tế hoặc hay bị self-biased. Cần tính toán chỉ số tương đồng với con người để đánh giá độ tin cậy câu trả lời.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric           | Threshold | Lý do                                                                                                      |
| ---------------- | --------: | ---------------------------------------------------------------------------------------------------------- |
| Faithfulness     |    >= 0.8 | Trong hệ thống CSKH thì trả lời liên quan chính sách thì nếu bị hallucination sẽ gây rủi ro pháp lý lớn    |
| Answer Relevance |    >= 0.7 | Đảm bảo bot trả lời đúng trọng tâm và không vòng vo                                                        |
| Completeness     |    >= 0.7 | Một số câu hỏi có nhiều cách trả lời hoặc có nhiều bước thì tiêu chí đảm bảo user không phải hỏi nhiều lần |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

Trước khi deploy code/prompt thì cần chạy offline evaluation trên golden dataset để phát hiện sớm regression, đo lường metric ragas, cosine tự động với chi phí thấp. Online evaluation chạy trên production với lưu lượng người dùng thực tế qua tracing hay logging. Cần thiết để theo dõi latency, user feedback, fallback rate. Human review để audit chất lượng thực tế, các edge case phức tạp và cập nhật lại golden dataset hay calibrate.

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
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục                      | Kết quả |
| ----------------------------- | ------- |
| Tổng số records               | 20 / 20 |
| Easy                          | 5 / 5   |
| Medium                        | 7 / 7   |
| Hard                          | 5 / 5   |
| Adversarial                   | 3 / 3   |
| Source documents được sử dụng | 10 / 10 |
| Validator status              | PASS    |

**Ba case đại diện cho quyết định thiết kế**

| ID  | Difficulty                     | Source document(s)                    | Vì sao case phù hợp với difficulty/attack type?                                                                                               |
| --- | ------------------------------ | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| E04 | Easy                           | `05_returns_and_exchanges.md`         | Fact lookup một điều kiện đơn về thời hạn trả thiết bị chưa mở. Expected answer có thể đối chiếu trực tiếp với evidence nguyên văn.           |
| H04 | Hard                           | `09_escalation_and_policy_updates.md` | Cần suy luận theo ngày đặt hàng và ngoại lệ membership: đơn trước 01/09/2026 dùng Policy 1.0 và không được kéo dài bởi OrbitPlus về sau.      |
| A02 | Adversarial / prompt injection | `00_system_scope.md`                  | Câu hỏi cố ép assistant tiết lộ hidden prompt, credentials và dữ liệu khách khác; attack type phù hợp với quy tắc chống instruction override. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

Khó nhất là giữ expected answer không rộng hơn evidence, đặc biệt với policy có điều kiện và ngày hiệu lực. Với H04, cần phân biệt ngày trao đổi policy với ngày đặt hàng; với returns và warranty, cần giữ riêng thời hạn, mốc bắt đầu và ngoại lệ. Vì vậy contexts được trích nguyên văn từ đúng source_doc, còn expected answer chỉ tổng hợp các claim mà evidence thực sự hỗ trợ.

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

| ID  | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
| --- | ---------------- | ---------: | ------------: | -----------: | --------: | -----------: | ------: | ------- | ------------ |
| E01 |                  |            |               |              |           |              |         |         |              |
| E02 |                  |            |               |              |           |              |         |         |              |
| E03 |                  |            |               |              |           |              |         |         |              |
| E04 |                  |            |               |              |           |              |         |         |              |
| E05 |                  |            |               |              |           |              |         |         |              |
| M01 |                  |            |               |              |           |              |         |         |              |
| M02 |                  |            |               |              |           |              |         |         |              |
| M03 |                  |            |               |              |           |              |         |         |              |
| M04 |                  |            |               |              |           |              |         |         |              |
| M05 |                  |            |               |              |           |              |         |         |              |
| M06 |                  |            |               |              |           |              |         |         |              |
| M07 |                  |            |               |              |           |              |         |         |              |
| H01 |                  |            |               |              |           |              |         |         |              |
| H02 |                  |            |               |              |           |              |         |         |              |
| H03 |                  |            |               |              |           |              |         |         |              |
| H04 |                  |            |               |              |           |              |         |         |              |
| H05 |                  |            |               |              |           |              |         |         |              |
| A01 |                  |            |               |              |           |              |         |         |              |
| A02 |                  |            |               |              |           |              |         |         |              |
| A03 |                  |            |               |              |           |              |         |         |              |

**Aggregate Report**

- Overall pass rate: \_\_\_\_%
- Avg Context Recall: \_\_\_\_
- Avg Context Precision: \_\_\_\_
- Avg Faithfulness: \_\_\_\_
- Avg Relevance: \_\_\_\_
- Avg Completeness: \_\_\_\_
- Failure type distribution: \_\_\_\_

**Ba cases có Overall Score thấp nhất**

1. ID: \_**\_ | Score: \_\_** | Failure type: \_\_\_\_
2. ID: \_**\_ | Score: \_\_** | Failure type: \_\_\_\_
3. ID: \_**\_ | Score: \_\_** | Failure type: \_\_\_\_

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> _Câu trả lời:_

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: \***\*\_\_\*\***

| Score | Tiêu chí domain-specific | Ví dụ response |
| ----: | ------------------------ | -------------- |
|     5 |                          |                |
|     4 |                          |                |
|     3 |                          |                |
|     2 |                          |                |
|     1 |                          |                |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
| --------- | ----------------- | --------------------- |
|           |                   |                       |
|           |                   |                       |
|           |                   |                       |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> _Câu trả lời:_

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí                  | Framework 1: \_\_\_\_ | Framework 2: \_\_\_\_ |
| ------------------------- | --------------------- | --------------------- |
| Setup complexity          |                       |                       |
| Metrics available         |                       |                       |
| CI/CD integration         |                       |                       |
| Kết quả trên cùng dataset |                       |                       |
| Insight rút ra            |                       |                       |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> _Phân tích:_

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID      | Recall before | Recall after | Precision before | Precision after | Delta Precision |
| ------- | ------------: | -----------: | ---------------: | --------------: | --------------: |
|         |               |              |                  |                 |                 |
|         |               |              |                  |                 |                 |
|         |               |              |                  |                 |                 |
|         |               |              |                  |                 |                 |
|         |               |              |                  |                 |                 |
| **Avg** |               |              |                  |                 |                 |

**Tại sao Recall dự kiến không đổi?**

> _Câu trả lời:_

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> _Câu trả lời:_

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
