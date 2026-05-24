---
title: Day 8 — RAG Pipeline (Truy xuất & Sinh câu trả lời)
sidebar_position: 8
---

# Day 8 — RAG Pipeline (Truy xuất & Sinh câu trả lời)

> **Mục tiêu**: Hiểu kiến trúc RAG đầy đủ, các kỹ thuật nâng cao (hybrid search, re-ranking, query rewriting, parent-doc retrieval), build pipeline production-ready.

## 1. RAG là gì? — Cấp độ cơ bản

**RAG** = **R**etrieval **A**ugmented **G**eneration. Ý tưởng:

1. **Retrieval**: lấy tài liệu liên quan từ kho riêng của bạn.
2. **Augmented**: nhét tài liệu vào prompt.
3. **Generation**: LLM sinh câu trả lời dựa trên tài liệu đó.

```
User question
     │
     ▼
┌──────────────────┐
│ Retrieval        │ ←── Vector DB (chunks đã embed từ Day 7)
└────────┬─────────┘
         │ top K chunks
         ▼
┌──────────────────┐
│ Augment Prompt   │ "Trả lời dựa trên context: {chunks}\nQ: {question}"
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ LLM Generation   │
└────────┬─────────┘
         │
         ▼
   Câu trả lời có nguồn
```

### 1.1 Vì sao cần RAG?

- LLM **không biết** dữ liệu private của bạn.
- LLM có **knowledge cutoff** — không biết thông tin mới.
- LLM **hallucinate** — RAG buộc trả lời "dựa trên" tài liệu.
- Cheaper than fine-tuning, dễ cập nhật kiến thức.

### 1.2 RAG mini code

```python
import chromadb
from anthropic import Anthropic

client = Anthropic()
chroma = chromadb.PersistentClient("./chroma_db")
col = chroma.get_collection("my_docs")

def rag(question: str, k: int = 4):
    hits = col.query(query_texts=[question], n_results=k)
    context = "\n\n".join(
        f"[{i+1}] {doc}" for i, doc in enumerate(hits["documents"][0])
    )
    prompt = f"""Dựa trên context sau, trả lời câu hỏi. Nếu không tìm thấy
trong context, nói "Tôi không có thông tin về điều này".

<context>
{context}
</context>

Câu hỏi: {question}
Trả lời (có trích nguồn [số]):"""

    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}],
    )
    return resp.content[0].text

print(rag("Doanh thu Q4 là bao nhiêu?"))
```

### Bài tập 8.1

Dùng vector DB ở Day 7 (50 file blog), implement function `rag(question)`. Hỏi 5 câu hỏi, check chất lượng câu trả lời.

---

## 2. Các vấn đề của Naive RAG — Cấp độ trung cấp

Naive RAG có rất nhiều **failure modes**:

| Vấn đề | Triệu chứng |
|--------|-------------|
| **Lost in the middle** | LLM bỏ qua context giữa khi context dài |
| **Bad chunking** | Câu trả lời bị cắt giữa 2 chunk |
| **Keyword vs semantic** | Câu hỏi có thuật ngữ chuyên ngành mà embedding không nắm |
| **Multi-hop fail** | Câu cần 2-3 bước suy luận, retrieval 1 lần không đủ |
| **No filter** | Retrieve cả tài liệu cũ, không liên quan |
| **Hallucinate ngoài context** | LLM vẫn bịa dù có context |

### 2.1 Hybrid Search: BM25 + Vector

**BM25** là thuật toán keyword cổ điển (cải tiến của TF-IDF). Khi query có **từ chuyên ngành / mã số / tên riêng**, BM25 thường tốt hơn vector.

→ **Hybrid**: chạy cả 2, merge điểm.

```python
# Pseudocode
bm25_scores  = bm25_index.search(query, k=20)
vec_scores   = vector_index.search(query, k=20)

# Reciprocal Rank Fusion
def rrf(rank_lists, k=60):
    scores = {}
    for ranks in rank_lists:
        for r, doc_id in enumerate(ranks):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + r + 1)
    return sorted(scores.items(), key=lambda x: -x[1])

merged = rrf([bm25_ids, vec_ids])
```

Hoặc dùng **Elasticsearch / OpenSearch / Qdrant** với hybrid native.

### 2.2 Re-ranking

Sau khi retrieve 20-50 chunk, dùng **re-ranker** (cross-encoder) để re-score chính xác hơn.

```
Stage 1: Bi-encoder (nhanh, recall cao, precision thấp)
  query → embed → search → top 50

Stage 2: Cross-encoder (chậm, precision cao)
  (query, chunk) → score → re-rank → top 5
```

Models phổ biến:
- `cohere-rerank-3.5` (API) — chất lượng cao
- `BAAI/bge-reranker-v2-m3` — open-source, multilingual

```python
import cohere
co = cohere.Client()

docs = [...]  # 50 chunks từ stage 1
ranked = co.rerank(model="rerank-3.5", query=question, documents=docs, top_n=5)
final_docs = [docs[r.index] for r in ranked.results]
```

### 2.3 Query Rewriting

User hỏi "Báo cáo Q4?" — quá mơ hồ. Rewrite thành "Báo cáo tài chính quý 4 năm 2025".

```python
def rewrite(query: str) -> list[str]:
    """Sinh 3 phiên bản query khác nhau."""
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=300,
        system="Tạo 3 reformulation cho query. Trả về 3 dòng.",
        messages=[{"role": "user", "content": query}],
    )
    return resp.content[0].text.splitlines()
```

Tìm bằng **mỗi rewrite**, merge kết quả (multi-query).

### 2.4 HyDE (Hypothetical Document Embeddings)

LLM "tưởng tượng" câu trả lời cho câu hỏi → embed câu tưởng tượng đó → search. Hữu ích khi query ngắn.

```python
hypo_answer = llm("Hãy trả lời câu hỏi sau: " + query)
hits = vector_db.search(embed(hypo_answer), k=5)
```

### Bài tập 8.2

1. Implement hybrid search (BM25 + vector) với [rank_bm25](https://pypi.org/project/rank-bm25/).
2. Thêm re-ranking với `bge-reranker`. Đo precision@5 trên 30 query có ground truth.
3. Implement query rewriting. So sánh recall.

---

## 3. RAG nâng cao — Cấp độ nâng cao

### 3.1 Parent Document Retrieval

Index **chunk con nhỏ** để search precise, nhưng trả lại **parent chunk lớn** cho LLM để có context.

```
Document (10K token)
  ├── Section 1 (2K token) ← parent
  │   ├── child chunk 1 (300t) ← embed & search
  │   ├── child chunk 2 (300t)
  │   └── ...
  └── Section 2 (2K token) ← parent
      └── ...

Query → match child chunk 1 → trả về Section 1 (2K) cho LLM.
```

LangChain có `ParentDocumentRetriever`. Hoặc tự code:

```python
def search_parent(query, k=3):
    children = vector_db.search(query, k=k)
    parent_ids = list({c.metadata["parent_id"] for c in children})
    parents = doc_store.get_many(parent_ids)
    return parents
```

### 3.2 Multi-hop / Iterative Retrieval

Câu hỏi: *"Vợ của CEO Apple sinh năm bao nhiêu?"*

- Bước 1: tìm "CEO Apple" → Tim Cook
- Bước 2: tìm "Tim Cook spouse" → ...

→ LLM tự reformulate query, retrieve lại đến khi đủ thông tin (giống ReAct).

### 3.3 Contextual Retrieval (Anthropic, 2024)

Thêm **context ngắn vào trước mỗi chunk** trước khi embed, giúp chunk "tự đứng" được.

```
Chunk gốc: "Doanh thu tăng 30% so với cùng kỳ."

Sau contextual:
"Đoạn này từ báo cáo Apple Q4 FY24, nói về mảng iPhone:
Doanh thu tăng 30% so với cùng kỳ."
```

Anthropic công bố giảm retrieval failure **49%** khi kết hợp Contextual + BM25 + Reranker.

```python
def contextualize(doc: str, chunk: str) -> str:
    prompt = f"""<document>{doc}</document>
<chunk>{chunk}</chunk>
Hãy viết 1-2 câu ngắn ngữ cảnh hoá chunk này trong document. Chỉ trả về câu đó."""
    resp = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=100,
        messages=[{"role": "user", "content": prompt}],
    )
    return resp.content[0].text + "\n" + chunk
```

Dùng **prompt caching** trên document để tiết kiệm chi phí (gọi cho hàng trăm chunk cùng 1 document).

### 3.4 Self-RAG / Corrective RAG

LLM tự đánh giá retrieval có liên quan không, có cần search lại không.

```
[Retrieve] → [Score relevance]
                   │
        ┌──────────┴──────────┐
     Relevant              Not relevant
        │                       │
     [Generate]            [Web search / Re-query]
```

### 3.5 GraphRAG (Microsoft, 2024)

Trích entity và relation từ tài liệu → build knowledge graph → retrieve theo subgraph. Hữu ích với câu hỏi "summary chủ đề rộng".

### Bài tập 8.3

1. Implement **contextual retrieval** trên 1 PDF dài. So sánh recall trước/sau khi contextualize.
2. Implement **parent document retriever**.
3. Đọc [Anthropic Contextual Retrieval blog](https://www.anthropic.com/news/contextual-retrieval). Replicate kết quả trên dataset của bạn.

---

## 4. RAG Production Pipeline

```
┌────────── Ingestion (offline) ──────────┐
│  PDF/HTML/MD → Parser → Chunker         │
│   → Contextualize → Embed → Vector DB   │
│   → BM25 Index → Metadata DB            │
└─────────────────────────────────────────┘

┌────────── Query (online) ───────────────┐
│  User Q → Query rewrite → Hybrid search │
│   → Re-rank → Top K → Format prompt     │
│   → LLM → Citation → Return             │
└─────────────────────────────────────────┘

┌────────── Observability ─────────────────┐
│  Log: query, retrieved chunks, score,   │
│       LLM output, latency, cost         │
│  Eval: regression, A/B, user feedback   │
└─────────────────────────────────────────┘
```

### 4.1 Citation & Hallucination check

Sau khi LLM trả lời, **verify** mỗi câu có nguồn trong context. Có thể dùng LLM khác chấm:

```
Câu trả lời: "Doanh thu Q4 đạt 120 tỷ."
Context: "...Quarterly revenue: 120B VND..."
→ Verified ✅
```

Hoặc dùng [Ragas](https://docs.ragas.io/) framework eval: faithfulness, answer relevancy, context recall, context precision.

### 4.2 Chi phí

```
Cost / query ≈
   (query rewrite tokens × $0.001)
 + (re-rank call: $1 / 1000 calls với Cohere)
 + (final LLM: ~3K tokens × $3/1M = $0.009)
 ≈ $0.01 - $0.05 / query
```

### 4.3 Latency budget

```
Embed query:       50 ms
Vector search:    100 ms
BM25:              30 ms
Re-rank top 50:   200 ms
LLM generate:    2000 ms (TTFT < 500ms với streaming)
Total:          ~2.5 s
```

---

## 5. Anti-patterns

:::warning
1. **K quá lớn**: lấy 30 chunk → LLM lost in middle, đắt tiền.
2. **Không có "I don't know"**: prompt phải có instruction "nếu không có context, hãy nói không biết".
3. **Skip eval**: thay model embedding, không re-test → recall giảm âm thầm.
4. **Mixed embedding model** trong cùng index.
5. **Không cache**: cùng query 100 lần/ngày — không cache là phí.
6. **Quên metadata filter**: trả về tài liệu của user khác (data leak).
:::

## 6. Tóm tắt

- RAG = **Retrieve + Augment + Generate**.
- Naive RAG **không đủ**: cần **hybrid (BM25 + vector)**, **re-rank**, **query rewrite**.
- **Contextual retrieval** (Anthropic) giảm fail tới 49%.
- **Parent-doc retriever**: search nhỏ, trả về to.
- Phải có **citation**, **"I don't know"**, **eval pipeline**.
- Quan tâm **latency, cost, observability** ngay từ đầu.

## 7. Bài tập tổng hợp

1. **Pipeline full**: Build RAG có: ingestion, hybrid search, re-rank, citation. Demo trên 50 tài liệu Markdown.
2. **Eval với Ragas**: Tạo eval set 30 (question, ground_truth, source). Chạy Ragas, đo faithfulness, context_precision, answer_relevancy.
3. **Compare retrievers**: vector-only vs hybrid vs hybrid+rerank vs hybrid+rerank+contextual. Bar chart 4 baselines.
4. **Đọc**:
   - [Anthropic — Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)
   - [Survey on RAG (2024)](https://arxiv.org/abs/2312.10997)
   - [Ragas docs](https://docs.ragas.io/)
5. **Real-world**: Build RAG chat trên tài liệu công ty bạn (intranet docs). Đo: bao nhiêu % câu hỏi user được giải quyết mà không cần hỏi đồng nghiệp?

---

> Hết Day 8. Mai sẽ học **Multi-Agent**, **MCP** (Model Context Protocol), **A2A**.
