---
title: Day 1 — AI & LLM Foundation
sidebar_position: 1
---

# Day 1 — AI & LLM Foundation

> **Mục tiêu**: Hiểu AI là gì, LLM (Large Language Model) hoạt động ra sao, các thành phần cốt lõi: token, embedding, kiến trúc Transformer, các pha huấn luyện.

## 1. AI là gì? — Cấp độ cơ bản

**AI (Artificial Intelligence)** là khả năng máy tính thực hiện những việc cần "trí thông minh" của con người: hiểu ngôn ngữ, nhận dạng hình ảnh, ra quyết định, học từ kinh nghiệm.

Các tầng lớp con của AI:

```
┌─────────────────────────────────────────────────┐
│ AI - Artificial Intelligence                     │
│ ┌─────────────────────────────────────────────┐ │
│ │ ML - Machine Learning                        │ │
│ │ ┌─────────────────────────────────────────┐ │ │
│ │ │ DL - Deep Learning                       │ │ │
│ │ │ ┌─────────────────────────────────────┐ │ │ │
│ │ │ │ LLM - Large Language Models          │ │ │ │
│ │ │ │ (GPT, Claude, Gemini, Llama, ...)   │ │ │ │
│ │ │ └─────────────────────────────────────┘ │ │ │
│ │ └─────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

- **ML (Machine Learning)**: Học từ dữ liệu, không lập trình rule cứng.
- **DL (Deep Learning)**: ML dùng mạng nơ-ron sâu (nhiều tầng).
- **LLM (Large Language Model)**: Mô hình DL chuyên về ngôn ngữ, có hàng tỷ tham số.

### Bài tập 1.1

1. Liệt kê 3 ví dụ AI bạn dùng hằng ngày (gợi ý: gợi ý của Netflix, Face ID, Google Maps...). Mỗi ví dụ thuộc tầng nào (ML/DL/LLM)?
2. Khác biệt giữa "AI tạo sinh" (Generative AI) và "AI phân biệt" (Discriminative AI) là gì?

---

## 2. LLM hoạt động thế nào? — Cấp độ trung cấp

LLM là một **mô hình xác suất**: cho một chuỗi token, nó dự đoán token tiếp theo có xác suất cao nhất.

### 2.1 Token — đơn vị xử lý

LLM không "đọc" chữ, mà đọc **token**. Token có thể là một từ, một mảnh từ, một dấu câu, hoặc thậm chí một byte.

Ví dụ với tokenizer của GPT:

```
Input: "Xin chào, AI!"
Tokens: ["Xin", " chào", ",", " AI", "!"]
Token IDs: [15234, 12876, 11, 9876, 0]
```

Code Python minh họa với `tiktoken`:

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o-mini")
text = "Xin chào, AI!"
tokens = enc.encode(text)
print(tokens)               # [15234, 12876, ...]
print(enc.decode(tokens))   # "Xin chào, AI!"
print(f"Số token: {len(tokens)}")
```

:::info Vì sao quan trọng?
- LLM tính **giá theo token** (input + output).
- LLM có **giới hạn context** tính theo token (vd: Claude Opus 4.7 = **1M token**).
- Ngôn ngữ khác nhau có hiệu suất token khác nhau (tiếng Việt thường tốn nhiều token hơn tiếng Anh).
:::

### 2.2 Embedding — biến token thành vector

Mỗi token được map sang một **vector số thực** ở không gian cao chiều (thường 768, 1536, 3072 chiều). Các token có nghĩa gần nhau thì vector cũng gần nhau (đo bằng cosine similarity).

```
"chó"   → [0.21, -0.08, 0.95, ..., 0.33]   (1536 chiều)
"mèo"   → [0.19, -0.05, 0.97, ..., 0.31]   ← gần "chó"
"xe"    → [-0.45, 0.72, 0.10, ..., -0.12]  ← xa "chó"
```

Code Python với OpenAI embedding:

```python
from openai import OpenAI
client = OpenAI()

resp = client.embeddings.create(
    model="text-embedding-3-small",
    input=["chó", "mèo", "xe hơi"],
)
vectors = [d.embedding for d in resp.data]

# Cosine similarity
import numpy as np
def cos(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

print("chó vs mèo:", cos(vectors[0], vectors[1]))  # ~ 0.7
print("chó vs xe:",  cos(vectors[0], vectors[2]))  # ~ 0.2
```

### 2.3 Kiến trúc Transformer (2017)

Tất cả LLM hiện nay đều dựa trên **Transformer** — kiến trúc do Google đề xuất trong paper *Attention Is All You Need* (2017).

```
        ┌──────────────────────────────┐
        │  Output: "trời"              │
        └──────────────▲───────────────┘
                       │
        ┌──────────────────────────────┐
        │  Softmax → token probabilities│
        └──────────────▲───────────────┘
                       │
        ┌──────────────────────────────┐
        │  Transformer Block × N        │
        │  ┌──────────────────────────┐ │
        │  │ Multi-Head Self-Attention│ │
        │  │ Feed Forward Network     │ │
        │  │ LayerNorm + Residual     │ │
        │  └──────────────────────────┘ │
        └──────────────▲───────────────┘
                       │
        ┌──────────────────────────────┐
        │ Embedding + Positional Encoding│
        └──────────────▲───────────────┘
                       │
              Input: "Hôm nay"
```

Hai phần quan trọng nhất:

- **Self-Attention**: cho phép mỗi token "nhìn" vào các token khác trong câu để hiểu ngữ cảnh.
- **Feed-Forward Network**: biến đổi phi tuyến để học pattern phức tạp.

**Attention** đại khái:

```
Attention(Q, K, V) = softmax( Q · K^T / sqrt(d_k) ) · V
```

Trong đó Q (Query), K (Key), V (Value) đều là các phép chiếu tuyến tính của embedding đầu vào.

### Bài tập 1.2

1. Cài `tiktoken`, đếm số token của 3 câu tiếng Việt và 3 câu tiếng Anh cùng nghĩa. Tỷ lệ chênh là bao nhiêu?
2. Embedding 5 từ: "vua", "hoàng hậu", "đàn ông", "phụ nữ", "máy tính". Tính cosine similarity giữa các cặp. Cặp nào gần nhất?
3. Vẽ lại sơ đồ Transformer trên giấy. Giải thích vì sao cần Positional Encoding.

---

## 3. Các pha huấn luyện LLM — Cấp độ nâng cao

Một LLM hiện đại trải qua **3 pha** chính:

### 3.1 Pre-training (tiền huấn luyện)

- Đọc **toàn bộ internet** (CommonCrawl, sách, code, Wikipedia...).
- Mục tiêu: dự đoán token tiếp theo (**next-token prediction**).
- Tốn rất nhiều: GPU/TPU cluster trong vài tháng, chi phí hàng chục triệu USD.
- Kết quả: **base model** — biết rất nhiều nhưng chưa biết "trò chuyện".

```python
# Pseudocode loss của pre-training
for batch in dataset:
    tokens = tokenizer(batch.text)
    for i in range(len(tokens) - 1):
        predicted = model(tokens[:i+1])
        loss = cross_entropy(predicted, tokens[i+1])
        loss.backward()
        optimizer.step()
```

### 3.2 Supervised Fine-Tuning (SFT)

- Dùng dataset (prompt, response) chất lượng cao do **con người viết**.
- Mục tiêu: dạy model trả lời theo định dạng mong muốn.
- Quy mô nhỏ hơn nhiều (10K-100K ví dụ).

Ví dụ một dòng SFT:

```json
{
  "prompt": "Giải thích Big-O notation cho người mới học?",
  "response": "Big-O dùng để mô tả tốc độ tăng trưởng của thuật toán..."
}
```

### 3.3 RLHF / DPO — Học từ phản hồi con người

- **RLHF** (Reinforcement Learning from Human Feedback): con người so sánh 2 câu trả lời, model học từ preference.
- **DPO** (Direct Preference Optimization): biến thể đơn giản hơn, không cần reward model riêng.

```
Prompt: "Viết hài về AI"
   ↓
Model sinh A: "Tại sao AI không kể chuyện cười? Vì nó luôn 'process' quá lâu."  ✅ chọn
Model sinh B: "AI không có khiếu hài hước."                                      ❌

→ Update model để xác suất sinh A > sinh B
```

:::warning Quan trọng
LLM **không có "kiến thức" cố định** — nó học **phân phối token** từ data. Nên nó có thể:
- **Hallucinate**: bịa ra thông tin sai, nhưng nghe rất tự tin.
- **Bias**: kế thừa định kiến trong dữ liệu huấn luyện.
- **Cutoff**: chỉ biết tới một thời điểm dữ liệu huấn luyện.
:::

### Bài tập 1.3

1. Tải một **base model** (vd: `meta-llama/Llama-3.1-8B`) và một **instruct model** (`Llama-3.1-8B-Instruct`). Hỏi cùng câu hỏi, so sánh output.
2. Tìm 1 ví dụ hallucination của ChatGPT/Claude và phân tích vì sao nó sai.
3. Đọc paper [InstructGPT](https://arxiv.org/abs/2203.02155) và tóm tắt 3 bước RLHF.

---

## 4. So sánh các LLM phổ biến (2025-2026)

| Model | Nhà sản xuất | Context | Mạnh ở |
|-------|--------------|---------|--------|
| Claude Opus 4.7 | Anthropic | 1M | Lý luận sâu, code, tool use |
| Claude Sonnet 4.6 | Anthropic | 200K | Cân bằng tốc độ/chất lượng |
| GPT-5 | OpenAI | 256K | Đa năng, multimodal |
| Gemini 2.5 Pro | Google | 2M | Video, multimodal |
| Llama 4 | Meta | 1M | Open-source, on-prem |
| Qwen 3 | Alibaba | 128K | Tiếng Á, mã nguồn mở |

:::tip Chọn model
- **Prototype nhanh**: Sonnet/GPT-4.1-mini (rẻ, nhanh).
- **Lý luận phức tạp**: Opus / GPT-5 / Gemini 2.5 Pro.
- **On-prem / privacy**: Llama / Qwen.
:::

---

## 5. Tóm tắt

- **AI ⊃ ML ⊃ DL ⊃ LLM**.
- LLM xử lý **token** (không phải chữ), token → **embedding** (vector).
- Kiến trúc lõi: **Transformer** với **self-attention**.
- 3 pha huấn luyện: **Pre-training → SFT → RLHF/DPO**.
- LLM có thể **hallucinate, bias, cutoff** — luôn cần kiểm chứng.
- Khi chọn model, cân nhắc: chất lượng, giá, context, latency, on-prem.

## 6. Bài tập tổng hợp

1. **Đọc tài liệu**: Đọc 1 trong 3 bài sau và tóm tắt 5 ý chính:
   - [Attention Is All You Need (2017)](https://arxiv.org/abs/1706.03762)
   - [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) — Jay Alammar
   - [State of GPT](https://www.youtube.com/watch?v=bZQun8Y4L2A) — Andrej Karpathy
2. **Hands-on**: Dùng OpenAI/Anthropic API, gửi 1 prompt với `temperature=0` và `temperature=1.0`, mỗi cái chạy 3 lần. So sánh độ ổn định.
3. **Khám phá**: Mở [Tokenizer Playground](https://platform.openai.com/tokenizer), nhập 1 đoạn tiếng Việt 100 từ. Đếm token. Thử dịch sang tiếng Anh — chênh bao nhiêu %?
4. **Thử thách**: Viết một chương trình Python dùng `tiktoken` để **đếm chi phí** ước tính của một file `.txt` khi gửi qua `gpt-4o-mini` (input $0.15/1M, output $0.60/1M token).

---

> Hết Day 1. Mai sẽ học cách **xác định bài toán nào nên dùng AI**.
