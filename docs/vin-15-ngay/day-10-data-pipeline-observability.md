---
title: Day 10 — Data Pipeline & Data Observability
sidebar_position: 10
---

# Day 10 — Data Pipeline & Data Observability

> **Mục tiêu**: Thiết kế data pipeline cho AI system (ingestion, transform, index, refresh), giám sát chất lượng dữ liệu, phát hiện drift sớm.

## 1. Data Pipeline cho AI — Cấp độ cơ bản

Hệ AI **chỉ tốt khi data tốt**. Pipeline xử lý data trong AI thường gồm:

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Sources  │───►│ Ingest   │───►│ Transform│───►│ Index/   │───►│ Serve    │
│ (DB, file│    │ (extract │    │ (clean,  │    │ Store    │    │ (RAG/LLM)│
│  API)    │    │  load)   │    │  chunk,  │    │ (vector, │    │          │
│          │    │          │    │  embed)  │    │  SQL)    │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
       ▲                                                              │
       └──────────────── feedback (logs, eval) ◄──────────────────────┘
```

### Các loại data nguồn

- **Structured**: SQL, NoSQL, CSV.
- **Semi-structured**: JSON, YAML, log.
- **Unstructured**: PDF, Word, HTML, ảnh, audio.
- **Stream**: Kafka, webhook event.

### ELT vs ETL

| | ETL (cũ) | ELT (mới, AI-friendly) |
|--|----------|------------------------|
| Order | Extract → Transform → Load | Extract → Load → Transform |
| Storage | Warehouse cuối | Lake (raw) → Warehouse |
| Linh hoạt | Khó re-transform | Re-transform thoải mái từ raw |
| Tool | Informatica, Talend | Airbyte, Fivetran, dbt |

### Bài tập 10.1

Vẽ data pipeline cho chatbot tra cứu nội quy công ty (data: PDF nội quy, sheet phòng ban, ticket lịch sử).

---

## 2. Build pipeline thực tế — Cấp độ trung cấp

### 2.1 Tool stack điển hình

| Layer | Tool phổ biến |
|-------|--------------|
| Ingest | Airbyte, Fivetran, Singer, Custom scripts |
| Storage (raw) | S3, GCS, Azure Blob — "data lake" |
| Storage (warehouse) | Snowflake, BigQuery, Redshift, DuckDB |
| Transform | dbt, Spark, Polars, Pandas |
| Orchestrate | Airflow, Dagster, Prefect, Temporal |
| Vector store | Chroma, Qdrant, Pinecone, pgvector |
| Monitoring | Great Expectations, Soda, Monte Carlo |

### 2.2 Mini pipeline với Prefect

```python
from prefect import flow, task
import chromadb
from anthropic import Anthropic

@task(retries=3)
def extract_pdfs(folder: str) -> list[str]:
    import glob
    return glob.glob(f"{folder}/*.pdf")

@task
def parse_pdf(path: str) -> dict:
    import pypdf
    reader = pypdf.PdfReader(path)
    text = "\n".join(p.extract_text() or "" for p in reader.pages)
    return {"path": path, "text": text}

@task
def chunk(doc: dict) -> list[dict]:
    text = doc["text"]
    chunks = [text[i:i+800] for i in range(0, len(text), 700)]
    return [{"source": doc["path"], "chunk": c} for c in chunks]

@task
def index(items: list[dict]):
    client = chromadb.PersistentClient("./db")
    col = client.get_or_create_collection("docs")
    col.add(
        ids=[f"{x['source']}-{i}" for i, x in enumerate(items)],
        documents=[x["chunk"] for x in items],
        metadatas=[{"source": x["source"]} for x in items],
    )

@flow
def ingest_pipeline(folder: str):
    paths = extract_pdfs(folder)
    docs = [parse_pdf(p) for p in paths]
    all_chunks = []
    for d in docs:
        all_chunks.extend(chunk(d))
    index(all_chunks)

if __name__ == "__main__":
    ingest_pipeline("./pdfs")
```

Chạy: `python pipeline.py`. Prefect cung cấp UI để xem flow, retry, schedule.

### 2.3 Schedule & incremental

- **Incremental**: chỉ xử lý file mới (`mtime > last_run`).
- **CDC** (Change Data Capture): theo dõi thay đổi trong DB.
- **Schedule**: cron hằng đêm, hoặc trigger khi có file mới (event-driven).

### 2.4 Idempotency

Pipeline phải **chạy lại an toàn**: cùng input → cùng output, không nhân đôi data.

→ Dùng `upsert` thay vì `insert`, có `idempotency key` (vd: hash content).

### Bài tập 10.2

1. Cài Prefect, chạy pipeline trên với 5 file PDF.
2. Sửa 1 PDF, chạy lại — pipeline có **chỉ re-index file đó** không? Nếu không, sửa để incremental.
3. Schedule chạy 6h sáng hằng ngày.

---

## 3. Data Observability — Cấp độ nâng cao

**Data observability** = monitor sức khoẻ data như monitor service.

### 3.1 5 Pillars của Data Observability (Monte Carlo)

| Pillar | Câu hỏi | Ví dụ metric |
|--------|---------|--------------|
| **Freshness** | Data có mới không? | Last update timestamp |
| **Volume** | Lượng data đúng không? | Row count, file size |
| **Distribution** | Phân phối hợp lý không? | Mean, stddev, histogram |
| **Schema** | Schema có thay đổi không? | Column count, types |
| **Lineage** | Đi từ đâu đến đâu? | Source → transforms → consumer |

### 3.2 Cho AI/LLM, thêm các pillar:

| Pillar | Câu hỏi |
|--------|---------|
| **Embedding drift** | Phân phối vector có đổi? |
| **Query distribution** | User hỏi gì khác trước không? |
| **Retrieval quality** | Recall@K còn ổn không? |
| **Output drift** | LLM trả lời khác kiểu trước không? |
| **Cost** | Chi phí token/query có tăng bất thường? |
| **Latency** | TTFT, total latency theo p50/p95/p99 |

### 3.3 Implementation với Great Expectations

```python
import great_expectations as gx

ctx = gx.get_context()
suite = ctx.add_expectation_suite("docs_quality")

# Định nghĩa expectations
suite.expect_column_values_to_not_be_null("text")
suite.expect_column_value_lengths_to_be_between("text", min_value=50, max_value=5000)
suite.expect_column_values_to_match_regex("source", r".*\.pdf$")

# Validate
batch = ctx.get_batch({"datasource_name": "my_docs", ...})
result = ctx.run_checkpoint("docs_checkpoint")
```

### 3.4 Drift detection cho embedding

```python
from scipy.stats import wasserstein_distance
import numpy as np

# Baseline: mean vector tuần trước
baseline_mean = np.load("baseline_mean.npy")
this_week = np.mean(embeddings_this_week, axis=0)

# Tính khoảng cách
drift = wasserstein_distance(baseline_mean, this_week)
if drift > THRESHOLD:
    alert("Embedding drift detected!")
```

### 3.5 Stack monitoring cho AI

```
┌──────────────────────────────────────────────┐
│ LLM/RAG App                                   │
│   └──► Logs (prompt, output, tokens, time)   │
│   └──► Metrics (Prometheus / OpenTelemetry)  │
│   └──► Traces (OTel — span theo agent step)  │
└──────────────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────────────┐
│ Observability platform                        │
│ - LangSmith / Langfuse / Helicone / Arize    │
│ - Grafana + Loki + Tempo                     │
│ - Datadog LLM Observability                  │
└──────────────────────────────────────────────┘
              │
              ▼
   Alerts (Slack/PagerDuty), Dashboards
```

### Bài tập 10.3

1. Cài [Langfuse](https://langfuse.com/) (open-source). Wrap LLM call để log trace.
2. Định nghĩa 3 alert: cost > $X/giờ, latency p99 > 5s, error rate > 5%.
3. Tạo dashboard Grafana với 4 panel: requests/min, error rate, p95 latency, cost.

---

## 4. Data Quality cho RAG

Trước khi index vào vector DB, check:

| Check | Cách |
|-------|------|
| Document **trùng lặp** | Hash MD5/SHA256, dedup |
| Chunk **quá ngắn** | `len(chunk) < 50` → bỏ |
| Encoding **lỗi** | utf-8 valid, kiểm regex |
| **PII** (số CMND, email cá nhân) | DLP scan, mask |
| **Outdated** | Có `expires_at`, auto-archive |
| **Language mismatch** | Detect language, group theo lang |
| **Toxic content** | Classifier (Perspective API, OpenAI moderation) |

```python
def quality_gate(doc: dict) -> bool:
    if len(doc["text"]) < 50:
        return False
    if not is_valid_utf8(doc["text"]):
        return False
    if contains_pii(doc["text"]):
        return False
    return True
```

## 5. Lineage tracking

Dùng [OpenLineage](https://openlineage.io/) chuẩn open: track flow từ source file → table → embedding → vector index → LLM output.

Lợi ích:
- Debug: "Tại sao chatbot trả về thông tin sai?" → trace ngược đến doc gốc.
- Compliance: "Data này có dùng cho AI training không?" → check lineage.
- Impact analysis: "Nếu xoá file X, hệ thống nào bị ảnh hưởng?" → trace forward.

---

## 6. Tóm tắt

- Pipeline AI = **Ingest → Transform → Index → Serve → Feedback**.
- Ưu tiên **ELT** + **incremental** + **idempotent**.
- Tool: Prefect/Dagster/Airflow + dbt + vector DB.
- Observability = **5 pillars** + AI-specific (embedding drift, query distribution).
- Quality gate trước khi index: dedup, PII scan, language, length, encoding.
- Trace lineage để debug và compliance.

## 7. Bài tập tổng hợp

1. **Pipeline production**: Build pipeline với Prefect/Dagster: ingest 1 nguồn (S3 / Notion / Google Drive) → quality gate → chunk → embed → ChromaDB. Có incremental + retry.
2. **Observability**: Cài Langfuse, log mọi LLM call. Tạo dashboard.
3. **Drift detection**: Mỗi tuần, tính trung bình vector của 1000 query mới. Alert khi Wasserstein distance > threshold.
4. **Đọc**:
   - [Monte Carlo — 5 Pillars](https://www.montecarlodata.com/blog-what-is-data-observability/)
   - [OpenLineage docs](https://openlineage.io/)
   - [Langfuse blog — LLM observability](https://langfuse.com/blog)
5. **Case study**: Khi Airbnb migrate sang LLM-based search, họ build pipeline nào? Đọc engineering blog của họ.

---

> Hết Day 10. Mai sẽ học **Guardrails, HITL, Responsible AI** — bảo vệ AI khỏi "đi hư".
