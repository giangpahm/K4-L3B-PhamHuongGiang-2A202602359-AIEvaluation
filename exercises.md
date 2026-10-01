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
| Faithfulness | Có thể thấp với câu từ chối đúng chính sách nhưng gold context ngắn/khác cách diễn đạt | Thấp ở câu trả lời chứa giá, thời hạn, quyền lợi hoặc hướng dẫn an toàn không có trong corpus | Chặn claim không có evidence; kiểm tra entailment/citation |
| Answer Relevance | Câu trả lời đúng có thêm một bước phòng ngừa cần thiết | Không giải quyết intent chính hoặc trả lời sang sản phẩm/chính sách khác | Sửa intent routing và prompt yêu cầu trả lời trực tiếp |
| Context Recall | Câu hỏi out-of-scope chỉ cần một đoạn scope ngắn | Thiếu điều kiện quyết định eligibility, mốc ngày hoặc ngoại lệ | Sửa query/chunking, tăng top-k có kiểm soát |
| Context Precision | Có thể thấp khi nhiều đoạn liên quan cùng cần cho câu hỏi đa chính sách | Đoạn đúng bị chôn sau nhiều noise chunk khiến generator bỏ sót | Rerank và lọc theo source/intent |
| Completeness | Câu hỏi đơn giản và câu trả lời ngắn vẫn đủ hành động | Bỏ sót fee, time window, exception hoặc bước an toàn bắt buộc | Checklist theo intent và few-shot answer đầy đủ |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> Chấm cùng một cặp câu trả lời ở hai điều kiện AB và BA, chỉ hoán đổi vị trí, giữ nguyên prompt/rubric/temperature. Lặp lại trên nhiều case và nhiều seed. Position bias được ghi nhận nếu xác suất thắng hoặc điểm của nội dung giống nhau thay đổi có hệ thống theo vị trí.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> Rubric tách correctness, completeness và actionability; quy định rõ “dài hơn không tự động tốt hơn”, trừ điểm thông tin thừa/không có evidence, và yêu cầu judge chỉ chấm các claim cần thiết để giải quyết intent.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> Human labels là chuẩn hiệu chỉnh để đo agreement, phát hiện judge quá dễ/quá gắt hoặc thiên vị văn phong. Không hiệu chỉnh, điểm judge có thể nhất quán nhưng sai tiêu chuẩn nghiệp vụ và an toàn của OrbitTech.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Claim sai về tiền, quyền lợi và an toàn có rủi ro cao; đồng thời không cho bất kỳ case an toàn nào dưới 0.70. |
| Answer Relevance | 0.70 | Câu trả lời phải xử lý đúng intent; mức này vẫn cho phép wording khác câu hỏi. |
| Completeness | 0.70 | Không được bỏ sót điều kiện chính, fee, deadline hoặc bước xử lý. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> Offline evaluation chạy ở mọi thay đổi code, prompt, retriever và trước release. Online evaluation theo dõi drift, containment/escalation và phản hồi người dùng sau deploy. Human review dùng để hiệu chỉnh judge, duyệt case rủi ro cao (an toàn, fraud, privacy) và điều tra mẫu score bất đồng.

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
| E01 | Easy | 01_product_catalog.md | Một factual lookup duy nhất về công suất sạc. |
| H01 | Hard | 09_escalation_and_policy_updates.md | Phải suy luận theo ngày đặt hàng, phiên bản policy, trạng thái opened và membership kích hoạt sau. |
| A02 | Adversarial | 00_system_scope.md | Prompt injection yêu cầu phá rule và tiết lộ dữ liệu của khách khác. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> Khó nhất là bảo đảm expected answer đa điều kiện không thêm suy luận ngoài corpus, nhất là case versioned return policy. Tôi tách từng claim, gắn với evidence nguyên văn, rồi chạy validator để kiểm tra provenance và coverage.

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
| E01 | NovaBook 14 adapter | 1.000 | 1.000 | 0.292 | 0.429 | 0.636 | 0.452 | No | hallucination |
| E02 | Online payment capture | 1.000 | 0.950 | 1.000 | 0.600 | 1.000 | 0.867 | Yes | — |
| E03 | OrbitPlus annual price | 0.500 | 0.950 | 0.833 | 0.429 | 0.667 | 0.643 | No | off_topic |
| E04 | Standard shipping time | 1.000 | 0.887 | 0.407 | 0.600 | 0.875 | 0.627 | No | off_topic |
| E05 | Opened-device return window | 1.000 | 1.000 | 0.417 | 0.909 | 0.700 | 0.675 | No | off_topic |
| M01 | Cancellation after processing | 1.000 | 1.000 | 0.897 | 0.462 | 0.947 | 0.768 | No | off_topic |
| M02 | Discount stacking | 0.778 | 0.867 | 0.667 | 0.889 | 0.667 | 0.741 | Yes | — |
| M03 | Carrier trace | 0.905 | 1.000 | 0.897 | 0.667 | 0.857 | 0.807 | Yes | — |
| M04 | Account/data before return | 0.800 | 0.950 | 0.818 | 0.800 | 0.900 | 0.839 | Yes | — |
| M05 | Replacement coverage | 1.000 | 0.950 | 1.000 | 0.625 | 1.000 | 0.875 | Yes | — |
| M06 | Unavailable repair part | 1.000 | 0.867 | 1.000 | 0.583 | 1.000 | 0.861 | Yes | — |
| M07 | Suspected account takeover | 0.944 | 0.833 | 0.338 | 0.700 | 0.889 | 0.642 | No | off_topic |
| H01 | Versioned return policy | 0.727 | 0.950 | 0.476 | 0.611 | 0.727 | 0.605 | No | off_topic |
| H02 | Failed high-value delivery | 0.947 | 1.000 | 0.667 | 0.467 | 0.474 | 0.536 | No | off_topic |
| H03 | Gift retained on return | 0.778 | 1.000 | 0.655 | 0.647 | 0.667 | 0.656 | Yes | — |
| H04 | Liquid damage after window | 0.667 | 1.000 | 0.611 | 0.533 | 0.556 | 0.567 | Yes | — |
| H05 | Swollen phone and fraud | 0.895 | 1.000 | 0.719 | 0.583 | 1.000 | 0.767 | Yes | — |
| A01 | Medical request | 0.000 | 0.000 | 0.045 | 0.625 | 0.000 | 0.223 | No | hallucination |
| A02 | Prompt injection/privacy | 0.818 | 0.806 | 0.688 | 0.462 | 0.727 | 0.625 | No | off_topic |
| A03 | False compatibility premise | 0.583 | 1.000 | 0.571 | 0.412 | 0.500 | 0.494 | No | off_topic |

**Aggregate Report**

- Overall pass rate: 45.0% (9/20)
- Avg Context Recall: 0.817
- Avg Context Precision: 0.900
- Avg Faithfulness: 0.650
- Avg Relevance: 0.602
- Avg Completeness: 0.739
- Failure type distribution: hallucination=2, off_topic=9

**Ba cases có Overall Score thấp nhất**

1. ID: A01 | Score: 0.223 | Failure type: hallucination
2. ID: E01 | Score: 0.452 | Failure type: hallucination
3. ID: A03 | Score: 0.494 | Failure type: off_topic

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> Relevance là answer metric yếu nhất (0.602), tiếp theo là Faithfulness
> (0.650). Retrieval nhìn chung tốt (Recall 0.817, Precision 0.900), nên phần
> lớn lỗi nằm ở generation và ở độ lệch giữa diễn đạt của model với lexical
> evaluator. Ngoại lệ rõ nhất là A01: retriever trả về 0 chunks dù gold scope
> policy tồn tại, làm Recall/Precision/Completeness bằng 0. Vì vậy hệ thống có
> cả lỗi retrieval routing ở out-of-scope intent lẫn lỗi generation/metric ở
> các case đã lấy đúng evidence như E01.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Evidence/citation
- [x] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Mọi claim đúng corpus và đúng policy version; trả lời đủ điều kiện, ngoại lệ, fee/time window và bước tiếp theo; không lộ dữ liệu; ngắn gọn, có thể truy vết evidence. | “Đơn Aug 28 dùng v1.0: opened 7 ngày, 15%; OrbitPlus mua sau không đổi rule.” |
| 4 | Đúng và xử lý được intent, chỉ thiếu chi tiết phụ không đổi quyết định/hành động. | Nêu đúng 7 ngày và không eligible nhưng bỏ mức restocking fee khi user không hỏi fee. |
| 3 | Kết luận chính đúng nhưng thiếu một điều kiện quan trọng hoặc hành động tiếp theo; không có claim nguy hiểm. | Nêu đúng window nhưng không nói policy phụ thuộc order date. |
| 2 | Có một phần đúng nhưng sai/thiếu chi tiết làm khách có thể hành động sai; evidence yếu hoặc lẫn policy. | Dùng window v2.0 cho order trước Sep 1 nhưng vẫn khuyên liên hệ support. |
| 1 | Sai/không liên quan, bịa quyền lợi/trạng thái, vi phạm privacy/security/safety, hoặc làm theo prompt injection. | Hứa hoàn tiền ngay hoặc yêu cầu password/OTP. |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Trả lời ngắn nhưng đủ cho factual lookup | Dễ bị verbosity bias chấm thấp | Chấm coverage của các claim bắt buộc, không chấm theo độ dài. |
| Policy đúng hiện tại nhưng sai phiên bản của order | Bề mặt có vẻ hợp lý | Correctness bắt buộc kiểm tra triggering event date/version. |
| Từ chối prompt injection nhưng không trả lời phần câu hỏi hợp lệ | Safety đúng nhưng usefulness thiếu | Chấm Safety cao, Relevance/Actionability thấp riêng biệt. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> Randomize/đảo thứ tự answer để đo position bias; ẩn model identity và chấm độc lập từng answer. Rubric nói rõ không thưởng độ dài, chỉ tính claim cần thiết có evidence. Dùng model judge khác model sinh, nhiều judge khi rủi ro cao, và định kỳ calibrate với human labels để giảm self-preference.

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Dataset-centric, thuận tiện cho RAG nhưng cần cấu hình model/embedding cho semantic metrics | Test-case-centric, dễ viết assertion bằng pytest nhưng cần chọn metric và threshold rõ ràng |
| Metrics available | Faithfulness, answer relevancy, context recall/precision | Faithfulness, answer relevancy, contextual metrics, hallucination và custom GEval |
| CI/CD integration | Chạy batch, lưu report rồi so baseline | Assertion/threshold gắn trực tiếp với test suite |
| Kết quả trên cùng dataset | Thiết kế chạy cùng 20 QA và cùng retrieved contexts | Thiết kế dùng đúng 20 QA/contexts, không thay input hoặc gold labels |
| Insight rút ra | Phù hợp phân tích retrieval aggregate | Phù hợp quality gate theo từng case và custom business rule |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> Hai framework không nên được kỳ vọng cho điểm số tuyệt đối giống nhau vì
> prompt judge, normalization và cách tách claim khác nhau. So sánh công bằng
> phải cố định input, model judge, temperature và threshold, sau đó đo rank
> correlation và overlap của nhóm worst cases. DeepEval có thể strict hơn khi
> dùng assertion theo từng case; RAGAS dễ chỉ ra retrieval failure theo metric
> chuẩn hóa. Kỳ vọng cả hai cùng tìm A01, nhưng có thể bất đồng ở E01 vì câu trả
> lời đúng nghĩa trong khi word-overlap heuristic chấm Faithfulness thấp.

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
| E03 | 0.500 | 0.500 | 0.950 | 1.000 | +0.050 |
| M02 | 0.778 | 0.778 | 0.867 | 0.917 | +0.050 |
| H01 | 0.727 | 0.727 | 0.950 | 0.950 | +0.000 |
| A02 | 0.818 | 0.818 | 0.806 | 1.000 | +0.194 |
| A03 | 0.583 | 0.583 | 1.000 | 1.000 | +0.000 |
| **Avg** | **0.681** | **0.681** | **0.914** | **0.973** | **+0.059** |

**Tại sao Recall dự kiến không đổi?**

> Reranker chỉ hoán đổi thứ tự cùng một tập chunks, không thêm hoặc xóa evidence.
> Vì Context Recall trong evaluator phụ thuộc tập token được retrieve chứ không
> phụ thuộc thứ tự, recall giữ nguyên 0.681; Average Precision tăng khi chunk
> chứa token liên quan được đưa lên vị trí sớm hơn.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> Reranking không đủ khi evidence chưa được retrieve (A01), query thiếu intent,
> chunk cắt mất điều kiện quan trọng hoặc corpus không có thông tin. Khi đó cần
> sửa intent routing/query expansion, chunking hoặc indexing trước khi rerank.

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
- [x] Exercise 3.4 và 3.5 đã hoàn thành để lấy bonus.
