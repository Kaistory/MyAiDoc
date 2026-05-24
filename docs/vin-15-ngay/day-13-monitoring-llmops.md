---
title: Day 13 — Monitoring & LLMOps
sidebar_position: 13
---

# Day 13 — Monitoring & LLMOps

> **Mục tiêu**: Hiểu LLMOps khác MLOps ra sao, build monitoring stack đầy đủ (metric, log, trace), alert hiệu quả, đo chất lượng output liên tục trong production.

## 1. LLMOps là gì? — Cấp độ cơ bản

**LLMOps** = MLOps cho LLM. Bao gồm:

- Quản lý prompt như code (version, test, deploy).
- Eval pipeline tự động.
- Theo dõi cost, latency, quality.
- Quản lý model version, fine-tune.
- Rollback, A/B test, canary release.

### So sánh MLOps vs LLMOps

| | MLOps (cổ điển) | LLMOps |
|--|----------------|--------|
| Asset chính | Model weights | **Prompt + model selection** |
| Train | Mỗi tháng/quý | Hiếm khi (mostly inference) |
| Eval | Accuracy, AUC | Faithfulness, relevance, toxic, cost |
| Drift | Data drift | **Prompt drift, model drift, user behavior drift** |
| Cost | GPU train | **Token API** (per-request) |
| Output | Số / class | **Text, JSON, agent action** |

### Bài tập 13.1

Liệt kê 5 metric bạn nghĩ là quan trọng nhất cho LLM app của mình. Vì sao chọn?

---

## 2. Observability Stack — Cấp độ trung cấp

### 2.1 Three pillars: Metrics, Logs, Traces

```
┌──────────────────────────────────────────────┐
│ Metrics (số, time-series)                     │
│ - request/sec, latency p50/p95/p99            │
│ - error rate, cost/hour                       │
│ - cache hit rate, eval score                  │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│ Logs (event, structured)                      │
│ - mỗi request: input, output, tokens, model  │
│ - errors, exceptions                         │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│ Traces (span theo flow)                       │
│ - span: rewrite → retrieve → rerank → LLM    │
│ - identify slow step                         │
└──────────────────────────────────────────────┘
```

### 2.2 Tool stack

**LLM-specific platforms** (recommend bắt đầu):
- [Langfuse](https://langfuse.com/) (open-source) ⭐
- [LangSmith](https://smith.langchain.com/) (LangChain ecosystem)
- [Helicone](https://www.helicone.ai/) (proxy-based)
- [Arize Phoenix](https://phoenix.arize.com/)
- [Datadog LLM Observability](https://www.datadoghq.com/)

**General observability**:
- Prometheus + Grafana
- OpenTelemetry (vendor-neutral)
- Honeycomb / New Relic / Datadog

### 2.3 Tích hợp Langfuse

```python
from langfuse.decorators import observe
from langfuse.openai import openai  # auto-trace

@observe()
def rag_query(question: str):
    chunks = retrieve(question)         # tự trace span
    resp = openai.chat.completions.create(
        model="gpt-4o",
        messages=[...],
    )
    return resp.choices[0].message.content
```

Mỗi call → tự gửi trace lên Langfuse: input, output, tokens, latency, cost.

### 2.4 OpenTelemetry cho LLM

```python
from opentelemetry import trace
tracer = trace.get_tracer("llm-app")

with tracer.start_as_current_span("rag") as span:
    span.set_attribute("user_id", uid)
    with tracer.start_as_current_span("retrieve"):
        chunks = retrieve(q)
    with tracer.start_as_current_span("generate"):
        out = llm(q, chunks)
    span.set_attribute("output_len", len(out))
```

Có chuẩn **OpenTelemetry GenAI Semantic Conventions** — attribute riêng cho LLM (model, tokens, temperature, ...).

### Bài tập 13.2

1. Cài Langfuse self-host (Docker compose). Tích hợp vào RAG app từ Day 8. Xem trace.
2. Tạo Grafana dashboard 4 panel: requests/min, p95 latency, cost/hour, error rate.

---

## 3. Quality Monitoring — Cấp độ nâng cao

Latency và cost dễ đo. **Chất lượng** khó hơn.

### 3.1 Online eval với LLM-as-judge

Mỗi N% request, gọi LLM khác chấm điểm:

```python
def judge(question, answer, context):
    prompt = f"""Cho câu hỏi, câu trả lời, và ngữ cảnh. Chấm 1-5 cho:
- groundedness: câu trả lời có dựa vào ngữ cảnh không?
- relevance: có trả lời đúng câu hỏi không?
- coherence: có rõ ràng, mạch lạc không?

Trả về JSON: {{groundedness:n, relevance:n, coherence:n, notes:""}}

Câu hỏi: {question}
Ngữ cảnh: {context}
Câu trả lời: {answer}"""
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=300,
        messages=[{"role": "user", "content": prompt}],
    )
    return json.loads(resp.content[0].text)
```

Sampling 5-10% production traffic → tránh chi phí gấp đôi.

### 3.2 User feedback signals

| Signal | Loại |
|--------|------|
| 👍 / 👎 | Explicit |
| User edit câu trả lời | Implicit negative |
| User hỏi lại tương tự | Implicit negative |
| User dừng giữa chừng | Implicit negative |
| Time to next message | Behavioral |
| Click vào citation | Positive engagement |

Lưu vào DB → train eval model riêng theo thời gian.

### 3.3 Drift detection

```python
# Daily job
this_week_queries = sample(query_log, n=1000, days=7)
last_month_queries = sample(query_log, n=1000, days=30)

# Embedding cluster
this_centroids = kmeans(embed(this_week_queries))
last_centroids = kmeans(embed(last_month_queries))

drift_score = wasserstein(this_centroids, last_centroids)
if drift_score > THRESHOLD:
    alert(f"Query distribution drift: {drift_score}")
```

### 3.4 Regression test khi đổi model/prompt

Trước khi deploy:

```python
# eval/golden_set.json
[
  {"q": "Doanh thu Q4?", "expected": "120 tỷ"},
  ...
]

def regression_test(new_prompt):
    score = 0
    for item in golden_set:
        out = llm(new_prompt, item["q"])
        if judge(item["expected"], out) >= 4:
            score += 1
    return score / len(golden_set)

if regression_test(new_prompt) < BASELINE - 0.03:
    raise BlockDeploy("Regression detected")
```

### Bài tập 13.3

1. Build LLM-as-judge module chấm 3 metric. Sampling 10% production traffic.
2. Setup user feedback collection (👍/👎) + lưu DB.
3. Tạo regression test với 30 golden cases. Run trong CI.

---

## 4. Cost Monitoring

### 4.1 Budget alert

```python
def track_cost(user_id, input_tokens, output_tokens, model):
    price = PRICING[model]  # {"input": 0.003, "output": 0.015} / 1K
    cost = (input_tokens * price["input"] + output_tokens * price["output"]) / 1000
    metric_cost.labels(model=model, user_id=user_id).inc(cost)
    return cost

# Daily budget alert
if daily_cost > DAILY_BUDGET * 0.8:
    slack_alert(f"⚠️ Cost ở {daily_cost/DAILY_BUDGET:.0%} budget")
```

### 4.2 Cost optimization

| Tactic | Tiết kiệm |
|--------|-----------|
| **Prompt caching** | 50-90% (long system prompt) |
| **Model routing** | Chuyển query đơn giản sang Haiku/mini | 80% cho subset |
| **Semantic cache** | Cache result cho query tương tự | 20-50% |
| **Shorter prompt** | Cắt few-shot không cần thiết | 10-30% |
| **Batch API** | OpenAI/Anthropic Batch giảm 50% | 50% |
| **Smaller model fine-tune** | Fine-tune Haiku thay vì prompt dài Opus | 50-80% |
| **Output token limit** | `max_tokens` thấp | tránh runaway |

### 4.3 Cost dashboard

Panels nên có:
- Cost/day theo model
- Cost/user (top 10)
- Cost/endpoint
- Token/request distribution
- Cache hit rate

---

## 5. Alerting

### 5.1 Symptom-based vs cause-based alerts

❌ Cause: "CPU > 80%" — alarm fatigue.
✅ Symptom: "p95 latency > 5s" hoặc "error rate > 2%" — actionable.

### 5.2 SLO/SLI framework

```
SLI: Service Level Indicator (đo)
  - availability = success / total
  - latency = p95 < 3s
  - quality = avg judge score > 4.0

SLO: Service Level Objective (mục tiêu)
  - 99% availability over 28d
  - 95% latency p95 < 3s
  - 90% quality > 4.0

Error budget = 1 - SLO. Khi cháy budget → freeze feature, fix tech debt.
```

### 5.3 Alert channels

- **Slack/Discord** cho warn
- **PagerDuty/Opsgenie** cho critical (page on-call)
- **Email** cho daily summary

### Bài tập 13.4

1. Define SLO cho 3 chỉ số: availability, latency, quality. Set realistic threshold.
2. Tạo alert trong Grafana / Langfuse khi vi phạm SLO.
3. Tính cost trung bình / query trong tuần qua. Top 3 user nào tốn nhất?

---

## 6. Incident Response

Khi LLM fail trong prod, quy trình chuẩn:

```
1. Detect    — alert fire
2. Triage   — đọc trace, identify root cause
3. Mitigate — quick fix (rollback prompt, switch model, kill switch)
4. Resolve  — fix gốc
5. Post-mortem — write doc, blameless, action items
```

### 5 Whys mẫu cho LLM incident

```
1. User phàn nàn câu trả lời sai về số liệu Q4.
   → Why? LLM hallucinate.
2. Why hallucinate? Context truyền vào không có số Q4.
3. Why không có? Retrieval chỉ lấy chunk Q3.
4. Why Q3? Sau khi đổi embedding model, recall giảm.
5. Why đổi mà không test? Không có regression test trên golden set.
→ Action: Always run eval gate trước deploy.
```

---

## 7. Tóm tắt

- LLMOps khác MLOps: **prompt là asset**, eval phức tạp, cost = token.
- 3 pillars: **metric, log, trace**.
- Tool: **Langfuse / LangSmith / Helicone / Datadog**.
- Quality: **LLM-as-judge** sampling, user feedback, regression test.
- Cost: **alert budget**, optimize bằng cache + routing + smaller model.
- Alert theo **symptom**, định nghĩa **SLO**, có **error budget**.
- Incident: detect → mitigate → resolve → blameless post-mortem.

## 8. Bài tập tổng hợp

1. **Full stack**: Tích hợp Langfuse vào app. Mọi LLM call có trace.
2. **Quality monitor**: LLM-as-judge 10% traffic, ghi vào Langfuse.
3. **Regression gate**: 30 golden cases, fail thì CI block.
4. **Cost dashboard**: Grafana panel cost/day, cost/user, cache hit rate.
5. **SLO**: Define SLO cho project. Track error budget tuần này.
6. **Đọc**:
   - [Google SRE book - SLO](https://sre.google/sre-book/service-level-objectives/)
   - [Langfuse docs](https://langfuse.com/docs)
   - [OpenTelemetry GenAI conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
7. **Mock incident**: Tự tạo 1 fail (đổi prompt làm hỏng JSON output). Đi qua quy trình detect → mitigate → post-mortem.

---

> Hết Day 13. Mai sẽ deep-dive **Evaluation Pipeline** và **Failure Analysis**.
