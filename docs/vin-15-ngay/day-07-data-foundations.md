---
title: Day 7 — Data Foundations (Embedding, Chunking, Vector Store)
sidebar_position: 7
---

# Day 7 — Data Foundations (Embedding, Chunking, Vector Store)

> **Mục tiêu**: Hiểu embedding, chia tài liệu (chunking) đúng cách, lưu/truy vấn bằng vector store (ChromaDB). Đây là nền tảng của RAG ở Day 8.

## 1. Embedding — Cấp độ cơ bản

**Embedding** = biến một đoạn text (câu, đoạn văn, tài liệu) thành **vector số thực**, sao cho các đoạn **gần nghĩa** thì vector **gần nhau** trong không gian.

```
"con mèo nằm ngủ"     ──► [0.21, -0.05, 0.92, ..., 0.31]
"chú mèo đang ngủ"    ──► [0.19, -0.06, 0.91, ..., 0.30]  ← rất gần
"giá Bitcoin hôm nay" ──► [-0.42, 0.71, 0.11, ..., -0.15] ← xa
```

Đo "gần xa" bằng **cosine similarity** (giá trị từ -1 đến 1, càng gần 1 càng tương đồng).

### Embedding models phổ biến

| Model | Chiều | Tiếng Việt | Giá |
|-------|-------|-----------|-----|
| OpenAI `text-embedding-3-small` | 1536 | ✅ tốt | $0.02/1M token |
| OpenAI `text-embedding-3-large` | 3072 | ✅ rất tốt | $0.13/1M token |
| Cohere `embed-v4` | 1024 | ✅ tốt | $0.10/1M token |
| Voyage `voyage-3` | 1024 | ✅ tốt | $0.06/1M token |
| `bge-m3` (open-source) | 1024 | ✅ tốt, hỗ trợ đa ngôn ngữ | free (self-host) |
| `sentence-transformers/all-MiniLM-L6-v2` | 384 | ⚠️ chỉ tiếng Anh | free |

### Code minh hoạ

```python
from openai import OpenAI
client = OpenAI()

texts = ["con mèo nằm ngủ", "chú mèo đang ngủ", "giá Bitcoin hôm nay"]
resp = client.embeddings.create(model="text-embedding-3-small", input=texts)
vecs = [d.embedding for d in resp.data]

import numpy as np
def cos(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

print(cos(vecs[0], vecs[1]))  # ~ 0.93
print(cos(vecs[0], vecs[2]))  # ~ 0.18
```

### Bài tập 7.1

1. Embed 10 câu tiếng Việt (5 về AI, 5 về nấu ăn). Vẽ ma trận similarity 10×10. Có thấy "cluster" rõ không?
2. So sánh `text-embedding-3-small` vs `bge-m3` (chạy local qua [sentence-transformers](https://www.sbge.net/)) trên cùng 10 câu — model nào phân cluster rõ hơn?

---

## 2. Chunking — Cấp độ trung cấp

Tài liệu thường rất dài (báo cáo 100 trang). Embedding cả tài liệu thì:

- **Mất chi tiết**: vector "trung bình" không đại diện cho phần cụ thể.
- **Khó retrieval**: query về phần X cũng match tài liệu Y vì nó dài.

→ Chia tài liệu thành **chunks** nhỏ, embed từng chunk.

### 2.1 Các chiến lược chunking

| Cách | Mô tả | Khi nào dùng |
|------|-------|--------------|
| **Fixed-size** | Cứ 500 token/chunk, có overlap 50 | Đơn giản, baseline |
| **Recursive** | Chia theo `\n\n` → `\n` → `.` → space | Mặc định trong LangChain |
| **Semantic** | Chia theo điểm nhảy nghĩa (embedding gap) | Chất lượng cao, đắt |
| **Sentence** | Mỗi chunk = N câu | Văn bản tự nhiên |
| **Structure-aware** | Theo heading H1/H2, table, code block | Markdown, code |
| **Late chunking** | Embed cả tài liệu trước, slice vector sau | Tài liệu rất ngắn (dưới 8K tokens) |

### 2.2 Code chunking đơn giản

```python
def chunk_text(text: str, size: int = 500, overlap: int = 50):
    chunks = []
    start = 0
    while start < len(text):
        end = start + size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks
```

### 2.3 Chunking thông minh với LangChain

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=120,
    separators=["\n## ", "\n### ", "\n\n", "\n", ". ", " "],
)
chunks = splitter.split_text(long_doc)
```

### 2.4 Tradeoff size

```
Chunk nhỏ (200-400 token):
  + Chính xác (vector tập trung 1 idea)
  + Retrieval precise
  - Mất context lân cận

Chunk lớn (1000-2000 token):
  + Có context xung quanh
  - Vector "trung bình", retrieval kém chính xác
```

:::tip Rule of thumb
- FAQ / Q&A: 200-400 token
- Technical doc: 600-1000 token, overlap 100
- Sách / long-form: 1200 token, overlap 200
- Code: chia theo function/class
:::

### 2.5 Metadata kèm chunk

Đừng quên gắn metadata: `source`, `page`, `section`, `date`, `tags`. Sau này filter rất tiện.

```python
chunks = [
    {
        "text": "...",
        "metadata": {
            "source": "report-q4.pdf",
            "page": 12,
            "section": "Financials",
            "date": "2025-Q4",
        }
    },
    ...
]
```

### Bài tập 7.2

1. Lấy 1 PDF dài 30 trang. Chia 3 cách: fixed-500, recursive, semantic. So sánh số chunk, độ dài trung bình.
2. Query "Doanh thu Q4 là bao nhiêu?" — cách nào trả về chunk **đúng** chứa số liệu?

---

## 3. Vector Store với ChromaDB — Cấp độ nâng cao

**Vector store** = database tối ưu cho tìm "vector gần nhất" (Approximate Nearest Neighbor, ANN).

### 3.1 Các vector DB phổ biến

| DB | Đặc điểm |
|----|----------|
| **ChromaDB** | Open-source, dễ self-host, embedded mode |
| **Qdrant** | Open-source, hiệu năng cao, Rust |
| **Weaviate** | Open-source, graph + vector |
| **Pinecone** | SaaS, managed, scale tốt |
| **Milvus** | Open-source, scale lớn |
| **pgvector** | Extension cho Postgres — quen thuộc |
| **Elasticsearch** | Hybrid keyword + vector |
| **FAISS** | Library (không phải DB), in-memory nhanh |

### 3.2 ChromaDB — quickstart

```bash
pip install chromadb
```

```python
import chromadb
from chromadb.utils import embedding_functions

client = chromadb.PersistentClient(path="./chroma_db")

embed_fn = embedding_functions.OpenAIEmbeddingFunction(
    api_key="sk-...",
    model_name="text-embedding-3-small",
)

collection = client.get_or_create_collection(
    name="my_docs",
    embedding_function=embed_fn,
)

# Add chunks
collection.add(
    ids=["doc1_p1", "doc1_p2", "doc2_p1"],
    documents=[
        "Doanh thu Q4 đạt 120 tỷ.",
        "Lợi nhuận sau thuế tăng 30%.",
        "AI giúp tự động hoá quy trình bán hàng.",
    ],
    metadatas=[
        {"source": "report.pdf", "page": 1},
        {"source": "report.pdf", "page": 2},
        {"source": "ai-note.pdf", "page": 1},
    ],
)

# Query
results = collection.query(
    query_texts=["Doanh thu là bao nhiêu?"],
    n_results=2,
    where={"source": "report.pdf"},  # filter metadata
)
print(results["documents"])
```

### 3.3 ANN index — vì sao nhanh?

Search exact qua hàng triệu vector quá chậm. ANN dùng cấu trúc index:

- **HNSW** (Hierarchical Navigable Small World) — phổ biến nhất, default của Chroma/Qdrant.
- **IVF** (Inverted File) — chia cluster trước, search trong cluster.
- **Product Quantization** — nén vector → tiết kiệm RAM.

```
HNSW concept:

        Layer 2 (sparse)
         ●─────●─────●
         │     │     │
        Layer 1
       ●─●─●─●─●─●
       │ │ │ │ │ │
        Layer 0 (dense, có tất cả vector)
      ●●●●●●●●●●●●●●●●●●●●

Query: tìm theo layer trên → xuống layer dưới (greedy).
Phức tạp O(log N) thay vì O(N).
```

### 3.4 Tuning recall vs speed

| Param | Tăng | Hiệu ứng |
|-------|------|---------|
| `ef_construction` | ↑ | Index chính xác hơn, build chậm |
| `ef_search` | ↑ | Recall cao hơn, query chậm |
| `M` | ↑ | Graph dày hơn, recall ↑, RAM ↑ |

### 3.5 Khi nào dùng pgvector?

- Bạn **đã có Postgres** trong stack.
- < 1M vector.
- Cần **transaction** (rollback, join với data cũ).

```sql
CREATE EXTENSION vector;
CREATE TABLE docs (
  id SERIAL PRIMARY KEY,
  content TEXT,
  embedding vector(1536)
);
CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops);

-- Query
SELECT content FROM docs
ORDER BY embedding <=> '[0.1,0.2,...]'::vector
LIMIT 5;
```

### Bài tập 7.3

1. Build mini-search: index 50 file `.md` từ blog cá nhân vào ChromaDB. Query "JavaScript closure" — top 3 có đúng không?
2. Thử cùng dataset với pgvector. So sánh độ phức tạp setup, hiệu năng.

---

## 4. Lỗi thường gặp

:::warning
1. **Cross-language**: embed bằng model chỉ tiếng Anh → query tiếng Việt fail. Chọn model multilingual (`bge-m3`, OpenAI v3).
2. **Stale data**: cập nhật chunk mà không re-index → vector cũ vẫn ở DB.
3. **Chunk quá nhỏ**: 100 token → vector noise.
4. **Không lưu metadata** → không filter được.
5. **Mix embedding model**: lưu vector từ 2 model khác nhau cùng collection → search vô nghĩa.
6. **Forget normalize**: một số DB cần vector đã `L2 normalize` mới tính cosine đúng.
:::

## 5. Embedding fine-tuning (nâng cao)

Nếu domain quá đặc thù (luật VN, y học, code Solidity...), embedding off-the-shelf chưa đủ:

- **Contrastive fine-tune**: cặp (query, positive_doc, negative_doc).
- Framework: [Sentence-Transformers](https://www.sbert.net/) MultipleNegativesRankingLoss.
- Dataset: 5K-50K cặp đủ để cải thiện 5-15% recall.

```python
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader

model = SentenceTransformer("intfloat/multilingual-e5-base")

train_examples = [
    InputExample(texts=["Doanh thu Q4?", "Q4 revenue was 120B VND"]),
    # ...
]

train_loader = DataLoader(train_examples, batch_size=16, shuffle=True)
loss = losses.MultipleNegativesRankingLoss(model)
model.fit([(train_loader, loss)], epochs=2)
model.save("./my-embed-model")
```

---

## 6. Tóm tắt

- **Embedding**: text → vector số. Đo similarity bằng **cosine**.
- **Chunking**: chia tài liệu, cân bằng **size vs context**. Luôn kèm **metadata**.
- **Vector DB**: lưu + ANN search. **ChromaDB** dễ bắt đầu, **pgvector** nếu đã có Postgres.
- **HNSW** là index mặc định — tradeoff recall/speed/RAM.
- Model multilingual nếu xử lý tiếng Việt (`bge-m3`, OpenAI v3).
- Fine-tune embedding khi domain quá đặc thù.

## 7. Bài tập tổng hợp

1. **Pipeline đầy đủ**: Cho thư mục 100 file `.txt`. Build pipeline: load → chunk (recursive 800/120) → embed → ChromaDB. Query "X" và in top 5 kết quả + nguồn.
2. **So sánh chunking**: Cùng dataset trên, thử fixed-500, recursive, semantic. Đo recall@5 trên 20 query đã có ground truth.
3. **Multilingual**: Tìm 1 dataset tiếng Việt (vd: tin VnExpress). Embed bằng OpenAI v3 và `paraphrase-multilingual-MiniLM-L12-v2`. So sánh.
4. **Đọc**:
   - [Embeddings — Vicki Boykis](https://vickiboykis.com/what_are_embeddings/)
   - [HNSW paper](https://arxiv.org/abs/1603.09320)
   - [Sentence-Transformers docs](https://www.sbert.net/)
5. **Quan sát**: Khi chunk có **overlap = 0** vs **overlap = 200**, retrieval khác nhau ra sao? Vẽ biểu đồ.

---

> Hết Day 7. Mai sẽ ráp toàn bộ thành **RAG Pipeline** với hybrid search và re-ranking.
