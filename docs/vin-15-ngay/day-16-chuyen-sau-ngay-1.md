---
title: Day 16 — Chuyên sâu Ngày 1 (Deep Dive AI/LLM)
sidebar_position: 16
---

# Day 16 — Chuyên sâu Ngày 1 (AI/LLM Foundation deep-dive)

> **Mục tiêu**: Đào sâu kiến thức Day 1. Từ pretraining → finetuning → alignment, hiểu sâu Transformer, attention, scaling laws, các kỹ thuật giảm chi phí (KV cache, quantization, MoE), tới các xu hướng mới (reasoning models, multimodal).

## 1. Transformer — sâu hơn

### 1.1 Self-attention từng bước

Cho chuỗi token có embedding `X` kích thước `n × d` (n token, mỗi token vector d chiều):

```
Step 1: Project X to Q, K, V
   Q = X · W_Q   (n × d_k)
   K = X · W_K   (n × d_k)
   V = X · W_V   (n × d_v)

Step 2: Compute scores
   scores = Q · K^T / sqrt(d_k)        (n × n)
   # scaling by sqrt(d_k) tránh softmax saturate khi d lớn

Step 3: Mask (cho decoder, causal)
   scores[i, j] = -inf if j > i

Step 4: Softmax row-wise
   attn = softmax(scores)              (n × n)

Step 5: Weighted sum
   output = attn · V                   (n × d_v)
```

```python
import torch, torch.nn.functional as F

def self_attention(X, W_Q, W_K, W_V, mask=None):
    Q = X @ W_Q
    K = X @ W_K
    V = X @ W_V
    d_k = K.size(-1)
    scores = Q @ K.transpose(-2, -1) / (d_k ** 0.5)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float("-inf"))
    attn = F.softmax(scores, dim=-1)
    return attn @ V
```

### 1.2 Multi-Head Attention

Thay vì 1 attention, dùng **h head song song**, mỗi head có ma trận chiếu `W_Q^i, W_K^i, W_V^i` riêng → concat lại.

```
MultiHead(X) = Concat(head_1, ..., head_h) · W_O
head_i = Attention(X · W_Q^i, X · W_K^i, X · W_V^i)
```

Mỗi head học một **loại quan hệ** khác nhau (syntactic, semantic, anaphoric...).

### 1.3 Positional Encoding

Vì attention **permutation-invariant**, cần "đánh dấu" vị trí.

#### Sinusoidal (Transformer gốc, 2017)

```
PE(pos, 2i)   = sin( pos / 10000^(2i/d) )
PE(pos, 2i+1) = cos( pos / 10000^(2i/d) )
```

#### RoPE (Rotary Position Embedding) — hiện đại

LLaMA, Mistral, Qwen dùng RoPE. Idea: **rotate** Q, K theo position trong không gian phức.

```
q'_m = R_m · q_m
k'_n = R_n · k_n
```

Trong đó `R_m` là ma trận xoay phụ thuộc vị trí `m`. Lợi: extrapolate tốt với chuỗi dài hơn training.

#### ALiBi (Attention with Linear Biases)

Thêm bias linear vào attention scores theo khoảng cách (i-j). Đơn giản, extrapolate cực tốt.

### Bài tập 16.1

Implement self-attention từ scratch trong Python (chỉ NumPy). Test với batch size 2, seq len 5, dim 8.

---

## 2. Pretraining sâu hơn

### 2.1 Objective function

LLM hiện đại train với **next-token prediction** (causal LM):

```
Loss = -(1/T) · Σ_{t=1..T}  log P( x_t | x_1, x_2, ..., x_{t-1} )
```

Cũng có **MLM** (BERT) — mask 15% token, dự đoán; nhưng LLM generative chủ yếu dùng causal.

### 2.2 Compute scaling — Chinchilla law

[Hoffmann et al. 2022](https://arxiv.org/abs/2203.15556) phát hiện: với compute fixed $C = 6ND$ (N=params, D=tokens), optimal **N và D cùng tăng tỷ lệ**.

```
Trước Chinchilla:     GPT-3 175B params, ~300B tokens     (under-trained!)
Sau Chinchilla:       70B params, ~1.4T tokens            (better!)
Hiện nay (LLaMA 3):   8-70B params, 15T tokens            (over-trained on data)
```

**Insight**: data đủ quan trọng như params.

### 2.3 Dataset

```
RedPajama / SlimPajama / FineWeb (open):
- CommonCrawl (web): 70%
- Code (GitHub): 5-10%
- Books, Wikipedia, ArXiv: 15-20%
- Math, reasoning: 5%
```

Chất lượng > số lượng. Dedup, filter spam, NSFW filter, license filter (Cao về copyright).

### 2.4 Tokenization

**BPE (Byte-Pair Encoding)**: bắt đầu từ byte/char, merge cặp xuất hiện nhiều → vocab 32K-128K.

```python
from transformers import AutoTokenizer
tok = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B")
print(tok.tokenize("Xin chào, AI!"))
# ['ĠX', 'in', 'Ġch', 'à', 'o', ',', 'ĠAI', '!']
```

**Tiktoken (OpenAI)**, **SentencePiece** (Google), **BPE-style** (LLaMA) — đều biến thể BPE.

> ⚠️ **Cảnh báo — Tiếng Việt**
>
> Vì tokenizer train chủ yếu trên tiếng Anh, tiếng Việt tốn **gấp 1.5-3x** token. Llama 3 đã cải thiện đáng kể.

### Bài tập 16.2

1. Train BPE tokenizer trên 1 corpus tiếng Việt 100MB bằng `tokenizers` library.
2. So sánh token count cho 100 câu tiếng Việt: tokenizer của bạn vs Llama 3 vs GPT-4o.
3. Đọc paper Chinchilla. Tính compute optimal cho 13B model.

---

## 3. Fine-tuning & Alignment sâu hơn

### 3.1 SFT (Supervised Fine-Tuning)

Cùng objective với pretrain, nhưng data là **(prompt, response)** chất lượng.

```python
# Pseudocode HuggingFace
from transformers import AutoModelForCausalLM, Trainer

model = AutoModelForCausalLM.from_pretrained("Llama-3.1-8B")
trainer = Trainer(
    model=model,
    train_dataset=instruction_dataset,
    args=TrainingArguments(
        learning_rate=2e-5,
        num_train_epochs=3,
        per_device_train_batch_size=4,
        gradient_accumulation_steps=16,
    ),
)
trainer.train()
```

### 3.2 LoRA / PEFT

Full fine-tune 70B model cần ~280GB VRAM. **LoRA** chỉ train 1-2% params (low-rank adapter) → 1 GPU 80GB đủ.

```
W' = W + B · A         với rank(B · A) = r << d
```

(`A` và `B` là 2 ma trận nhỏ rank thấp; chỉ train chúng, đóng băng `W`.)

```python
from peft import LoraConfig, get_peft_model

config = LoraConfig(
    r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05, bias="none", task_type="CAUSAL_LM",
)
model = get_peft_model(base_model, config)
print(model.print_trainable_parameters())  # ~0.1-1% trainable
```

**QLoRA**: combine 4-bit quantization + LoRA → fine-tune 70B trên 1 GPU 24GB.

### 3.3 RLHF — Reinforcement Learning from Human Feedback

3 bước:

```
1. SFT model M_SFT (đã có)
2. Train reward model R(x, y) → score
   - Data: cặp (y_w, y_l) — human chọn y_w hơn
   - Loss: -log σ(R(x, y_w) - R(x, y_l))
3. Optimize M_SFT bằng PPO/A2C, reward = R(x, y) - β·KL(M | M_SFT)
```

KL penalty giữ model không "drift" khỏi distribution tự nhiên.

### 3.4 DPO — Direct Preference Optimization

[Rafailov et al. 2023](https://arxiv.org/abs/2305.18290) — không cần reward model riêng, optimize trực tiếp từ preference pair:

```
Loss_DPO = -log σ( β · [ log(π_θ(y_w|x) / π_ref(y_w|x))
                       - log(π_θ(y_l|x) / π_ref(y_l|x)) ] )
```

Trong đó `y_w` là câu được chọn (preferred), `y_l` là câu bị bỏ; `π_θ` là model đang train, `π_ref` là reference (thường là SFT model). Đơn giản hơn RLHF rất nhiều, đang dần thay thế PPO trong nhiều pipeline.

### 3.5 Constitutional AI (Anthropic)

LLM **tự critique** + **tự revise** theo principles, không cần human label nhiều.

```
1. SFT model trả lời prompt → output_v1
2. LLM tự critique: "có vi phạm principles nào?"
3. LLM tự revise → output_v2
4. SFT trên output_v2
5. Optional: RLAIF (RL từ AI feedback)
```

### Bài tập 16.3

1. Fine-tune Llama 3.2 1B với LoRA trên 1 dataset 1000 cặp instruction-response.
2. So sánh trước/sau trên 20 câu hỏi.
3. Đọc paper DPO. Implement DPO trên cùng dataset.

---

## 4. Inference Optimization

### 4.1 KV Cache

Khi sinh token tiếp theo, không cần tính lại Q,K,V của các token cũ → cache K, V.

```
Naive: O(n²) flops cho mỗi token
KV cache: O(n) cho mỗi token mới

Cost: memory O(2 · L · n · d_model) — có thể rất lớn
```

### 4.2 Quantization

Giảm precision để giảm memory:

| Precision | Memory cho 70B model |
|-----------|---------------------|
| FP32 | 280 GB |
| FP16 / BF16 | 140 GB |
| INT8 | 70 GB |
| INT4 / Q4_K_M | ~40 GB |

Tools: bitsandbytes, GPTQ, AWQ, GGUF (llama.cpp).

```python
from transformers import AutoModelForCausalLM, BitsAndBytesConfig

bnb = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype="bfloat16",
)
model = AutoModelForCausalLM.from_pretrained("Llama-3.1-70B", quantization_config=bnb)
```

### 4.3 Flash Attention

Thuật toán attention re-arrange để **tránh load attention matrix** vào HBM. Tăng tốc 2-4x.

### 4.4 Speculative Decoding

Dùng model nhỏ "đoán" K token, model lớn verify trong 1 pass. Tăng tốc 2-3x.

### 4.5 Continuous Batching (vLLM)

Khác với static batching, vLLM merge requests mới vào batch đang chạy → throughput cao gấp 10x.

### 4.6 MoE (Mixture of Experts)

Mixtral, GPT-4 (rumored), DeepSeek dùng MoE: chỉ 1 phần experts active mỗi token → "effective params" thấp hơn total.

```
Mixtral 8x7B: 47B total params, ~13B active mỗi token.
DeepSeek-V3:  671B total,        37B active.
```

### Bài tập 16.4

1. Load Llama 3.1 8B với 4-bit quantization. So sánh memory, latency với FP16.
2. Cài vLLM, serve cùng model. Benchmark throughput 100 concurrent requests.
3. Đọc Flash Attention paper. Tóm tắt 3 trick chính.

---

## 5. Scaling Laws & Emergent abilities

### 5.1 Scaling laws

[Kaplan et al. 2020](https://arxiv.org/abs/2001.08361): loss giảm theo power law của compute, data, params.

```
L(C) ≈ (C_0 / C)^α
```

Không có "ceiling" rõ ràng — cứ scale là tốt hơn (cho tới khi data hết).

### 5.2 Emergent abilities

Một số khả năng **chỉ xuất hiện ở scale lớn**: arithmetic, in-context learning multi-step, code, ...

[Wei et al. 2022](https://arxiv.org/abs/2206.07682). Sau này bị tranh cãi (xem [Schaeffer et al. 2023](https://arxiv.org/abs/2304.15004)) — có thể chỉ là artifact của metric.

### 5.3 Mixture & specialization

Hiện đang chuyển dịch: thay vì 1 model siêu lớn, dùng **specialized models** + router:

```
Query → Router → Math model
                ↘ Code model
                ↘ Reasoning model
                ↘ General chat model
```

OpenAI o-series, Claude extended thinking, DeepSeek-R1 — tách reasoning thành phase riêng.

---

## 6. Multimodal LLM

### 6.1 Vision-Language Models (VLM)

GPT-4V, Claude Vision, Gemini, LLaVA: image → encoder (vd: CLIP/SigLIP) → token-like embeddings → fed vào LLM.

```
Image (224x224)
    │ ViT encoder
    ▼
patches embeddings (256 × d)
    │ projection
    ▼
"visual tokens" (256 × d_llm)
    │
    ▼
LLM (treats as prefix tokens)
```

### 6.2 Audio / Video

Whisper (speech → text), AudioLM (audio gen), Sora (text → video).

### 6.3 Any-to-any (Omni)

GPT-4o, Gemini 2.5: 1 model native cho text+image+audio.

### Bài tập 16.5

1. Dùng Claude Vision API, gửi 1 ảnh hoá đơn, trích structured data.
2. Cài LLaVA-1.6 7B local, thử với 5 ảnh.
3. Đọc paper [Flamingo](https://arxiv.org/abs/2204.14198) hoặc [LLaVA](https://arxiv.org/abs/2304.08485).

---

## 7. Reasoning Models (o1, o3, R1, Claude extended thinking)

### 7.1 Idea

Thay vì trả lời ngay, model "nghĩ" trong block ẩn (chain-of-thought dài) trước. Có thể dùng tới hàng chục nghìn token thinking.

```
User: "Giải hệ phương trình..."
        │
        ▼
Model thinking (hidden):
  "Let me set up equations...
   Try substitution...
   Wait, that's wrong, try again..."
        │
        ▼ (after N thinking tokens)
Final answer: "x=3, y=2"
```

### 7.2 Training

- RL với reward = correctness trên math/code/logic tasks.
- DeepSeek-R1 open-source recipe: SFT + RL trên problems có verifiable answer.

### 7.3 Trade-off

| | Standard model | Reasoning model |
|--|----------------|-----------------|
| Latency | 1-3s | 10-60s |
| Cost / query | Thấp | Cao (× thinking tokens) |
| Math, logic | Yếu | Mạnh |
| Chat thường | Tốt | Over-think |

→ Use case: research, math, code khó. Không cần với chat đơn giản.

### Bài tập 16.6

So sánh `claude-sonnet-4-6` (standard) vs `claude-opus-4-7` extended thinking trên 10 bài toán logic. Đo accuracy, latency, cost.

---

## 8. Roadmap học tiếp

```
Tuần 1-2: Implement Transformer from scratch (nanoGPT - Karpathy)
Tuần 3-4: Fine-tune Llama 3 1B với LoRA + DPO
Tuần 5-6: Build vLLM inference server, benchmark
Tuần 7-8: Reasoning model — replicate part of DeepSeek-R1 recipe
Tuần 9+:  Specialize (Vision, Audio, Agent...)
```

## 9. Tóm tắt

- Transformer = embedding + multi-head self-attention + FFN. Positional encoding (Sinusoidal → RoPE → ALiBi).
- Pretraining objective: next-token prediction. Chinchilla law: cân bằng N và D.
- Alignment: SFT → RLHF/DPO → Constitutional AI.
- Inference opt: **KV cache, quantization, Flash Attention, speculative decoding, MoE**.
- Scaling laws: loss giảm power-law theo compute.
- Multimodal: image/audio → visual/audio tokens → LLM.
- Reasoning models: thinking phase ẩn, RL trên verifiable reward.

## 10. Bài tập tổng hợp

1. **nanoGPT**: Đi qua [Karpathy nanoGPT](https://github.com/karpathy/nanoGPT). Hiểu từng dòng code. Train tiny model trên Shakespeare.
2. **Fine-tune**: LoRA fine-tune Llama 3.2 1B trên 1 task riêng (vd: text-to-SQL Việt Nam). Đo cải thiện.
3. **Quantization**: Convert model về GGUF 4-bit bằng llama.cpp. Chạy trên Mac/CPU.
4. **Reasoning replication**: Đọc DeepSeek-R1 paper. Thử SFT một small model với chain-of-thought trace.
5. **Đọc kỹ**:
   - [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
   - [Chinchilla](https://arxiv.org/abs/2203.15556)
   - [LoRA](https://arxiv.org/abs/2106.09685)
   - [DPO](https://arxiv.org/abs/2305.18290)
   - [Constitutional AI](https://arxiv.org/abs/2212.08073)
   - [Flash Attention](https://arxiv.org/abs/2205.14135)
   - [DeepSeek-R1](https://arxiv.org/abs/2501.12948)
6. **Viết blog**: Tóm tắt 1 trong các paper trên thành bài blog Việt Nam (~1500 từ), share lên dev.to / Medium.

---

> 🎉 Hoàn thành lộ trình Vin 16 ngày! Bạn đã có nền tảng vững chắc từ **prompt** tới **production**, từ **inference** tới **research**. Hành trình AI thực sự mới chỉ bắt đầu. Hãy **build, ship, đo, lặp lại**.
