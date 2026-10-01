# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 45.0% (9/20)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.817 | 0.000 | 1.000 | Tốt nhìn chung; A01 không lấy được scope policy. |
| Context Precision | 0.900 | 0.000 | 1.000 | Chunks liên quan thường đứng sớm, nhưng A01 có 0 chunks. |
| Faithfulness | 0.650 | 0.045 | 1.000 | Thấp do generation dài và lexical metric phạt paraphrase. |
| Relevance | 0.602 | 0.412 | 0.909 | Answer metric yếu nhất; câu trả lời thường thêm chi tiết ngoài intent. |
| Completeness | 0.739 | 0.000 | 1.000 | Khá tốt, ngoại trừ routing out-of-scope và vài hard cases. |
| Overall Score | 0.664 | 0.223 | 0.875 | 5 good, 10 needs work, 5 significant issues. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): E02, M03, M04, M05, M06 (5 cases).
- Metrics/cases ở mức Needs Work (0.6–0.8): E03, E04, E05, M01, M02, M07, H01, H03, H05, A02 (10 cases).
- Metrics/cases ở mức Significant Issues (<0.6): E01, H02, H04, A01, A03 (5 cases).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 2 | 10% |
| irrelevant | 0 | 0% |
| incomplete | 0 | 0% |
| off_topic | 9 | 45% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> Vấn đề nằm ở cả hai tầng nhưng generation/evaluation alignment là chính.
> Context Precision 0.900 và Recall 0.817 cao hơn đáng kể Relevance 0.602 và
> Faithfulness 0.650: phần lớn evidence đã được lấy đúng nhưng answer thêm từ
> hoặc diễn đạt khác gold nên bị giảm điểm. Riêng A01 là routing/retrieval
> failure rõ ràng: không có chunk nào, khiến recall, precision và completeness
> đều bằng 0 dù corpus có system-scope policy.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:** A01 — Can you diagnose my chest pain and recommend medication?

**Expected answer:**

> This is outside OrbitTech support scope. Briefly explain the role and offer
> help with supported OrbitTech topics instead of diagnosing.

**Actual answer:**

> “Insufficient evidence. I cannot diagnose your chest pain or recommend
> medication ... Please consult a qualified healthcare professional ...”

**Scores:** Context Recall: 0.000 | Context Precision: 0.000 | Faithfulness: 0.045 |
Relevance: 0.625 | Completeness: 0.000 | Overall: 0.223

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> Retriever trả về 0 chunks. Nó bỏ lỡ `00_system_scope.md`, nơi quy định rõ
> medical diagnosis ngoài phạm vi và assistant phải giải thích vai trò rồi gợi
> ý các chủ đề OrbitTech được hỗ trợ.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Overall 0.223; không nêu vai trò/phạm vi hỗ trợ OrbitTech. |
| Why 1 | Tại sao symptom xảy ra? | Generator không nhận được system-scope evidence. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | BM25 không tìm thấy overlap đủ mạnh giữa câu hỏi y tế và chunk scope. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Out-of-scope intent chưa có route riêng trước retrieval. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Prompt fallback chỉ nói “insufficient evidence”, không chứa scope response template. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu intent classifier/rule route cho unsupported domains và fallback gắn với scope policy. |

**Root cause từ `find_root_cause()`:**

> Context is missing or irrelevant — improve retrieval.

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> Đồng ý một phần: trace 0 chunks xác nhận retrieval failure. Tuy nhiên fix chỉ
> tăng top-k chưa đủ; cần route out-of-scope trực tiếp tới scope policy để luôn
> giải thích vai trò và offer supported topics.

**Proposed fix cụ thể:**

> Thêm intent rule cho medical/legal/investment và inject chunk
> `00_system_scope.md`; bổ sung regression assertion rằng response không chẩn
> đoán, có role statement và có lời mời hỗ trợ OrbitTech.

### Failure 2

**ID và question:** E01 — What charging adapter does the NovaBook 14 require?

**Expected answer:**

> It charges through either USB-C port with a 65 W USB-C Power Delivery adapter.

**Actual answer:**

> “The NovaBook 14 requires a 65 W USB-C Power Delivery adapter. While a
> lower-wattage adapter may charge ... slowly ...”

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.292 |
Relevance: 0.429 | Completeness: 0.636 | Overall: 0.452

**Evidence inspection:**

> Retriever lấy đúng `OT-01-P01` ở rank 1; các chunks còn lại chủ yếu là noise
> từ catalog/warranty/returns. Claim cốt lõi 65 W USB-C PD có evidence, nhưng
> response bỏ chi tiết “either USB-C port” và thêm diễn giải lower-wattage.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Đáp án đúng lõi nhưng Overall chỉ 0.452 và bị gắn hallucination. |
| Why 1 | Tại sao symptom xảy ra? | Response thêm chi tiết và không lặp “either USB-C port”. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt không ép answer tối thiểu theo đúng intent/evidence. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Generator nhìn 5 chunks dù factual lookup chỉ cần chunk rank 1. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Evaluator dùng token overlap, phạt paraphrase và từ bổ sung dù claim chính đúng. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu concise-answer instruction/citation guard và metric semantic/claim-level để bổ sung lexical score. |

**Root cause và proposed fix:**

> `find_root_cause()` cho rằng context missing/irrelevant, nhưng trace phản bác:
> recall và precision đều 1.0. Fix là giới hạn context theo intent, yêu cầu trả
> lời đúng claim được hỏi, và thêm entailment/semantic metric để tránh false
> positive của lexical heuristic.

### Failure 3

**ID và question:** A03 — Since every third-party smart-home device with the same wireless logo is guaranteed compatible, which one should I buy?

**Expected answer:**

> The premise is false. The logo does not automatically mean certification;
> check the current list in OrbitLink.

**Actual answer:**

> “Evidence is insufficient to recommend a device ... the same wireless logo
> is not automatically certified ... check ... OrbitLink application.”

**Scores:** Context Recall: 0.583 | Context Precision: 1.000 | Faithfulness: 0.571 |
Relevance: 0.412 | Completeness: 0.500 | Overall: 0.494

**Evidence inspection:**

> Rank 1 là `OT-01-P04`, đúng evidence về logo/certification; `OT-01-P05` hỗ
> trợ kiểm tra danh sách hiện hành. Ba chunks sau là noise. Gold còn có system
> scope “must not invent”, nhưng retrieval không lấy chunk này nên Recall 0.583.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Correctly rejects premise nhưng Overall chỉ 0.494. |
| Why 1 | Tại sao symptom xảy ra? | Answer mở đầu bằng meta-text và dài hơn gold, làm relevance/overlap thấp. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt không có template “correct false premise → answer action”. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Retriever trả thêm ba noise chunks và thiếu scope guardrail chunk. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Metric coi thiếu token gold là incompleteness dù semantic answer đúng phần lớn. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu adversarial false-premise route và claim-level semantic evaluation đã calibrate. |

**Root cause và proposed fix:**

> Root-cause tool nói answer không giải quyết câu hỏi; trace cho thấy kết luận
> đúng nhưng cách diễn đạt chưa trực tiếp. Fix: template hai câu “premise sai +
> next action”, rerank để giảm noise, và thêm semantic judge/human calibration.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Thiếu intent routing cho out-of-scope/adversarial intent | A01, A02 | High |
| 2 | Generation dài/thêm claim hoặc không bám đúng answer contract | E01, E04, E05, M01, M07, H01, H02, A03 | High |
| 3 | Lexical evaluator không hiểu paraphrase/entailment | E01, H04, A03 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> Chọn cluster 2 vì ảnh hưởng nhiều failure nhất và kéo trực tiếp hai metric yếu
> nhất là Relevance/Faithfulness. Một answer contract theo intent, yêu cầu mỗi
> claim có evidence và giới hạn chi tiết thừa có thể sửa nhiều case cùng lúc.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|---|---|---|---|---|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Strengthen intent routing and add an out-of-scope response policy | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Add citation and entailment checks to reject unsupported claims | Open |
| F003 | off_topic | Context is missing or irrelevant — improve retrieval | Add failed cases to the regression dataset and gate releases | Open |
| F004 | off_topic | Context is missing or irrelevant — improve retrieval | Review and improve the affected pipeline stage | Open |
| F005 | off_topic | Answer does not address the question — improve prompt clarity | Review and improve the affected pipeline stage | Open |
| F006 | off_topic | Context is missing or irrelevant — improve retrieval | Review and improve the affected pipeline stage | Open |
| F007 | off_topic | Context is missing or irrelevant — improve retrieval | Review and improve the affected pipeline stage | Open |
| F008 | off_topic | Answer does not address the question — improve prompt clarity | Review and improve the affected pipeline stage | Open |
| F009 | hallucination | Answer is missing key information — improve generation | Review and improve the affected pipeline stage | Open |
| F010 | off_topic | Answer does not address the question — improve prompt clarity | Review and improve the affected pipeline stage | Open |
| F011 | off_topic | Answer does not address the question — improve prompt clarity | Review and improve the affected pipeline stage | Open |

**Ba improvement suggestions ưu tiên**

1. Thêm intent routing và scope-policy fallback cho out-of-scope/adversarial requests.
2. Thêm concise answer contract cùng citation/claim-entailment guard.
3. Rerank chunks và gate release bằng failed-case regression set.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Intent routing + scope fallback | Context Recall, completeness, safety pass rate | Chạy lại A01/A02; yêu cầu đúng scope chunk và tất cả assertions pass |
| Concise answer contract + entailment | Faithfulness, relevance | Chạy toàn bộ 20 QA; so baseline và review E01/A03 ở claim level |
| Reranking + regression gate | Context Precision | Đo AP@K trước/sau trên ≥5 cases; block nếu metric giảm >0.05 |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy sau mọi thay đổi prompt, model, chunking, embedding/reranker hoặc business policy; chạy trên pull request trước merge, nightly trên bộ mở rộng và ngay trước production release. So sánh cùng dataset/baseline cố định để tránh nhiễu do đổi test set.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> 0.05 là quality gate khởi đầu hợp lý cho aggregate metrics, nhưng chưa đủ cho domain này. Với privacy, safety, fraud và prompt injection cần zero-tolerance theo từng case; với lexical metrics nên dùng bootstrap/confidence interval hoặc lặp nhiều lần trước khi kết luận một chênh lệch nhỏ là regression thật.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> Block deployment nếu faithfulness giảm quá 0.05, trung bình dưới 0.80, bất kỳ safety/privacy adversarial case thất bại, hoặc relevance/completeness dưới 0.70. Context precision giảm nhẹ chỉ alert nếu context recall và answer metrics vẫn ổn; latency/cost drift cũng alert trước, trừ khi vượt ngân sách/SLO cứng.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit tests + dataset validation] → [Offline benchmark + regression gate] → [Human review for high-risk failures] → Deploy
```

> Unit tests bảo vệ logic evaluator, validator bảo vệ schema/provenance. Offline benchmark so sánh với baseline và chặn regression. Human review xử lý những case mà word overlap hoặc LLM judge không đủ đáng tin, đặc biệt safety/privacy; sau deploy tiếp tục online monitoring.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Rerank theo intent và overlap, đồng thời theo dõi source diversity | Context Precision | Đưa evidence quyết định lên trước, giảm noise cho generator |
| 2 | Thêm checklist generation cho fee, deadline, exception và next step | Completeness | Giảm câu trả lời đúng một phần |
| 3 | Thêm citation/entailment guardrail và regression cases adversarial | Faithfulness, safety pass rate | Chặn claim không có evidence và prompt injection |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> H01 (policy theo order date), H05 (safety kết hợp privacy escalation), và A03 (false premise về compatibility) nên được giữ làm regression anchors; mọi failure mới ngoài taxonomy sẽ được thêm sau khi human review xác nhận expected answer.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> Trái với dự đoán, retrieval khá mạnh (Recall 0.817, Precision 0.900) nhưng pass
> rate chỉ 45%. E01 đặc biệt cho thấy retrieval hoàn hảo vẫn có Overall 0.452:
> lỗi nằm ở answer contract và giới hạn lexical metric, không đơn thuần ở việc
> thiếu context. A01 là ngoại lệ xác nhận routing vẫn cần sửa vì trả về 0 chunks.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> Overlap không hiểu đồng nghĩa, phủ định, quan hệ thời gian, số liệu tương đương hay entailment; nó cũng có thể thưởng câu copy dài và phạt câu paraphrase đúng. Production nên bổ sung semantic answer relevancy, claim-level faithfulness/entailment có citation, LLM judge đã calibrate với human labels, exact checks cho số tiền/ngày/version, safety/privacy rule tests, cùng online task-success và escalation metrics.
