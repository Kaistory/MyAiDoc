---
title: Day 14 — Evaluation Pipeline & Failure Analysis
sidebar_position: 14
---

# Day 14 — Evaluation Pipeline & Failure Analysis

> **Mục tiêu**: Xây eval pipeline đáng tin cậy, hiểu các metric chuẩn (BLEU, ROUGE, F1, BERTScore, LLM-as-judge), phân tích lỗi có hệ thống (error analysis).

## 1. Vì sao eval là việc khó nhất? — Cấp độ cơ bản

LLM trả về **text tự do**. Khác với classifier (đúng/sai rõ ràng), đánh giá text:

- Một câu trả lời đúng có **nhiều cách viết** ("120 tỷ" vs "120B VND").
- Đúng **ngữ pháp** chưa chắc đúng **nội dung**.
- "Hay" là chủ quan.
- Có hallucination khó phát hiện tự động.

→ Cần **eval pipeline có hệ thống** thay vì "test bằng cảm tính".

### Golden Set vs Production Eval

```
Golden Set (offline)              Production Eval (online)
- Curated bởi expert              - Sample từ real traffic
- Ground truth chính xác           - Không có ground truth
- Dùng trước deploy                - Dùng để monitor drift
```

### Bài tập 14.1

Bạn đang build chatbot tư vấn thiết bị y tế. Liệt kê 10 câu hỏi sẽ làm golden set, ai sẽ duyệt ground truth?

---

## 2. Metrics phổ biến — Cấp độ trung cấp

### 2.1 Reference-based (so với ground truth)

| Metric | Đo | Hạn chế |
|--------|-----|---------|
| **Exact Match** | Khớp 100% chữ | Quá khắt khe |
| **F1 token** | Overlap token | Bỏ qua nghĩa |
| **BLEU** | N-gram overlap (translation) | Không hiểu nghĩa |
| **ROUGE** | Recall n-gram (summarization) | Tương tự |
| **BERTScore** | Semantic similarity bằng BERT | Phụ thuộc embedding |
| **Semantic similarity** | Cosine với sentence embedding | Vẫn không hoàn hảo |

```python
# F1 token-level
def f1(pred, ref):
    p_set = set(pred.split())
    r_set = set(ref.split())
    if not p_set or not r_set:
        return 0
    inter = p_set & r_set
    p = len(inter) / len(p_set)
    r = len(inter) / len(r_set)
    return 2*p*r / (p+r) if (p+r) else 0
```

### 2.2 Reference-free (LLM-as-judge)

Dùng LLM khác chấm. Phổ biến nhất hiện nay.

```python
JUDGE_PROMPT = """Chấm điểm câu trả lời sau theo thang 1-5:
- 5: chính xác, đầy đủ, mạch lạc
- 4: chính xác, hơi thiếu
- 3: phần đúng phần sai
- 2: chủ yếu sai
- 1: hoàn toàn sai

Câu hỏi: {q}
Câu trả lời: {a}
Ground truth: {gt}

Trả về: {{"score": n, "reason": "..."}}"""

def llm_judge(q, a, gt, model="claude-sonnet-4-6"):
    resp = client.messages.create(
        model=model, max_tokens=200,
        messages=[{"role": "user", "content": JUDGE_PROMPT.format(q=q, a=a, gt=gt)}],
    )
    return json.loads(resp.content[0].text)
```

:::warning Position bias
Khi so 2 câu trả lời A vs B, LLM judge **thiên về câu A** (vị trí đầu). Mẹo: shuffle vị trí, hoặc chấm độc lập.
:::

### 2.3 RAG-specific (Ragas)

| Metric | Đo |
|--------|-----|
| **Faithfulness** | Câu trả lời có dựa trên context không? |
| **Answer relevancy** | Có trả lời đúng câu hỏi không? |
| **Context precision** | Top-K chunk có thật sự liên quan? |
| **Context recall** | Có thiếu chunk quan trọng không? |
| **Answer similarity** | Semantic giống ground truth bao nhiêu? |

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

dataset = {"question": [...], "answer": [...], "contexts": [...], "ground_truth": [...]}
result = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_precision])
print(result)
```

### 2.4 Agent-specific

| Metric | Đo |
|--------|-----|
| **Task success rate** | % task hoàn thành |
| **Step efficiency** | Số step trung bình / task |
| **Tool accuracy** | % tool call đúng |
| **Cost / task** | Token spent |
| **Recovery rate** | Tự sửa lỗi được bao nhiêu % |

### Bài tập 14.2

1. Tạo golden set 30 mẫu cho RAG app từ Day 8.
2. Implement Ragas eval. So sánh trước/sau khi thêm re-ranking.
3. Implement LLM-as-judge với position-bias mitigation.

---

## 3. Eval Pipeline trong CI — Cấp độ nâng cao

### 3.1 Pipeline chuẩn

```
┌──────────────┐
│ Code/Prompt  │
│ change       │
└──────┬───────┘
       ▼
┌──────────────┐
│ Unit tests   │ ← các tool, retrieval logic
└──────┬───────┘
       ▼
┌──────────────┐
│ Eval golden  │ ← 100-1000 cases
│ set          │
└──────┬───────┘
       ▼
┌──────────────┐
│ Compare vs   │ Quality drop > 3%?
│ baseline     │
└──────┬───────┘
   pass │ fail
       ▼     ▼
   Deploy  Block
```

### 3.2 Eval framework structure

```
eval/
  golden/
    qa_50.jsonl
    rag_30.jsonl
    agent_20.jsonl
  metrics/
    f1.py
    judge.py
  runners/
    run_qa.py
    run_rag.py
  reports/
    baseline.json
    latest.json
```

```python
# runners/run_rag.py
import json
from app import rag

def run(golden_file: str):
    results = []
    for item in load_jsonl(golden_file):
        pred = rag(item["question"])
        scores = {
            "faithfulness": llm_judge_faith(pred, item["contexts"]),
            "relevance":    llm_judge_relevance(pred, item["question"]),
            "f1":           f1(pred, item["answer"]),
        }
        results.append({"id": item["id"], "pred": pred, **scores})
    return results

if __name__ == "__main__":
    out = run("eval/golden/rag_30.jsonl")
    json.dump(out, open("eval/reports/latest.json", "w"))
```

### 3.3 CI gate

```yaml
# .github/workflows/eval.yml
- name: Run eval
  run: python eval/runners/run_rag.py

- name: Compare baseline
  run: |
    python eval/compare.py \
      --baseline eval/reports/baseline.json \
      --latest   eval/reports/latest.json \
      --threshold 0.03
```

```python
# eval/compare.py
base = json.load(...); curr = json.load(...)
for metric in ["faithfulness", "relevance", "f1"]:
    b = mean(item[metric] for item in base)
    c = mean(item[metric] for item in curr)
    diff = c - b
    print(f"{metric}: {b:.3f} → {c:.3f} (Δ {diff:+.3f})")
    if diff < -threshold:
        exit(1)
```

### Bài tập 14.3

Build eval pipeline đầy đủ cho RAG app:
1. Golden set 30 case.
2. 3 metric: F1, faithfulness, relevance.
3. CI gate fail nếu < baseline - 3%.

---

## 4. Failure Analysis có hệ thống

Khi eval chỉ ra "accuracy 78%" — chưa hữu ích. Cần biết **22% sai vì sao**.

### 4.1 Phân loại lỗi

Tạo taxonomy:

```
Failure modes:
├─ Retrieval
│  ├─ Wrong chunk (recall miss)
│  ├─ Irrelevant chunk (precision miss)
│  └─ Chunk cắt giữa câu trả lời
├─ Generation
│  ├─ Hallucination (bịa)
│  ├─ Refusal ("I don't know" sai)
│  ├─ Format sai (JSON parse fail)
│  ├─ Off-topic
│  └─ Citation sai
├─ Edge case
│  ├─ Câu hỏi mơ hồ
│  ├─ Đa ngôn ngữ
│  └─ Code-switching
└─ System
   ├─ Timeout
   ├─ Rate limit
   └─ Parse error
```

Mỗi sample fail → tag vào 1+ category.

### 4.2 Error matrix

```
                    Hallucination  Refusal  Bad retrieval  Format
Math question            5%          2%        10%          1%
Multi-hop                15%         5%        20%          0%
Out-of-domain            2%         50%         0%          0%
...
```

→ Insight: multi-hop yếu vì retrieval. Refusal phần lớn là out-of-domain (đúng!).

### 4.3 Quy trình error analysis

```
1. Lấy 50-100 fail cases từ eval/production.
2. Tự đọc, không dựa LLM (ít nhất lần đầu).
3. Tag thủ công vào taxonomy.
4. Tính tỷ lệ → top 3 lỗi.
5. Fix theo Pareto: top 1 lỗi → cải thiện rõ rệt nhất.
6. Re-run eval, đo cải thiện.
7. Lặp lại.
```

:::tip
Đừng nhảy vào "tuning model" trước khi đọc 100 fail bằng mắt. 80% insight nằm ở việc đọc thật cẩn thận một số ít sample.
:::

### 4.4 Tool: Phoenix

[Arize Phoenix](https://phoenix.arize.com/) (open-source) — UI tag fail, cluster, drill-down.

### Bài tập 14.4

1. Lấy 50 fail từ eval. Tự đọc, tag taxonomy.
2. Tính top 3 lỗi. Đề xuất fix cho top 1.
3. Implement fix, re-run eval. Cải thiện bao nhiêu %?

---

## 5. A/B Testing prompt/model

### 5.1 Split traffic

```python
import random

def chat(q):
    # 50/50 split
    variant = "B" if random.random() < 0.5 else "A"
    if variant == "A":
        out = llm(q, model="claude-sonnet-4-6", prompt=PROMPT_V1)
    else:
        out = llm(q, model="claude-sonnet-4-6", prompt=PROMPT_V2)
    log_event(variant, q, out)
    return out
```

### 5.2 Đo kết quả

Sau 1-2 tuần, đo:
- Quality (judge score, user 👍/👎).
- Latency, cost.
- Business metric (conversion, retention).

Statistical significance: dùng [Sequential testing](https://www.evanmiller.org/sequential-ab-testing.html) hoặc Bayesian.

:::warning Không A/B test quá nhiều thứ cùng lúc
Thay đổi 1 thứ một lần. Nếu đổi prompt + model + top-K cùng lúc → không biết yếu tố nào gây ra cải thiện.
:::

---

## 6. Tóm tắt

- Eval AI khó vì **output text tự do**.
- Dùng kết hợp **reference-based** (F1, BERTScore) + **LLM-as-judge** + **user feedback**.
- **Ragas** cho RAG: faithfulness, relevance, context precision/recall.
- Pipeline trong CI có **golden set**, **eval gate**, **threshold**.
- **Failure analysis** thủ công đầu tiên, **taxonomy** rõ ràng.
- **A/B test** chỉ 1 yếu tố mỗi lần.

## 7. Bài tập tổng hợp

1. **Golden set**: Build 50 case cho RAG / chatbot của bạn. Có ground truth.
2. **Ragas eval**: Chạy Ragas, report 4 metric.
3. **CI gate**: GitHub Actions chạy eval, fail nếu < baseline.
4. **Failure analysis**: Tag 50 fail, vẽ pie chart taxonomy. Đề xuất fix.
5. **A/B test**: Soạn 2 prompt khác nhau. Split 50/50 trong 1 ngày, tổng kết.
6. **Đọc**:
   - [Eugene Yan — LLM evals](https://eugeneyan.com/writing/llm-evaluators/)
   - [Hamel Husain — Evals are the next bottleneck](https://hamel.dev/blog/posts/evals/)
   - [Ragas docs](https://docs.ragas.io/)
   - [Phoenix docs](https://docs.arize.com/phoenix)

---

> Hết Day 14. Mai là ngày cuối — tổng kết về **triển khai Agent thực tế** và chi phí.
