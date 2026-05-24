---
title: Day 15 — Triển Khai Agent Thực Tế, Chi Phí & Định Hướng
sidebar_position: 15
---

# Day 15 — Triển Khai Agent Thực Tế, Chi Phí Vận Hành & Định Hướng

> **Mục tiêu**: Tổng hợp tất cả 14 ngày, đi qua case study triển khai agent thực tế end-to-end, tính chi phí vận hành chuẩn, định hướng nghề nghiệp & học tập tiếp theo.

## 1. Tổng kết hành trình 14 ngày

```
Foundation       │ Day 1-2:  Hiểu LLM, biết khi nào dùng AI
Agent & Prompt   │ Day 3-4:  ReAct, prompt engineering, tool calling
Product          │ Day 5-6:  PRD, risk, prototype
Data & RAG       │ Day 7-8:  Embedding, RAG pipeline
Multi-Agent      │ Day 9:    Supervisor, MCP, A2A
Operations       │ Day 10-13: Pipeline, observability, guardrails, deploy
Quality          │ Day 14:   Eval, failure analysis
Production       │ Day 15:   Triển khai thực tế (today)
```

## 2. Case Study: Build Agent từ 0 đến Production

### Bối cảnh
*"Trợ lý hỗ trợ khách hàng nội bộ cho công ty SaaS 50 nhân viên, ~2000 ticket/tháng. Giảm 50% ticket lặp lại do thông tin đã có trong docs."*

### Step 1 — Discovery (1 tuần)

- Phỏng vấn 5 support agent, đọc 300 ticket cuối tháng.
- Phân loại: 60% câu hỏi lặp lại có docs giải đáp.
- Estimate value: 1000 ticket/tháng × 15p mỗi cái × $20/h = **$5000/tháng tiết kiệm**.

→ Go.

### Step 2 — PRD (1 tuần)

- User: support agent (assist mode) + end customer (auto mode).
- Success: deflection rate > 30%, CSAT không giảm.
- Risk register: hallucination, PII leak, fallback khi không biết.
- Eval set: 100 câu hỏi từ ticket lịch sử.

### Step 3 — Prototype (2 tuần)

```
Stack chọn:
- Frontend: existing Intercom plugin
- Backend: Python FastAPI
- LLM: Claude Sonnet 4.6 (chất lượng + cost)
- RAG: ChromaDB (start) → migrate Qdrant nếu cần
- Observability: Langfuse
- Cloud: GCP Cloud Run (đã có sẵn account)
```

Build:
- Ingest 200 docs từ Notion + Help Center.
- Chunk recursive 800 token, overlap 120.
- Embed `text-embedding-3-small`.
- Hybrid search (BM25 + vector) + Cohere re-rank.
- Prompt template với "I don't know" fallback.

Eval 100 cases:
- Faithfulness: 4.2 / 5
- Relevance: 4.5 / 5
- F1 token: 0.62
- → Pass baseline ban đầu.

### Step 4 — Beta (2 tuần)

- 5 support agent dùng assist mode (chatbot suggest, human send).
- Thu thập 👍/👎, edit suggestion.
- Build feedback loop: bad suggestions → review → cải thiện docs hoặc prompt.

Tuần 1: deflection 22% (thấp). Đọc 50 fail → top lỗi: chunking cắt code snippet giữa câu → fix structure-aware chunking.

Tuần 2: deflection 35%. CSAT giữ nguyên.

### Step 5 — Production (1 tuần)

- Auto mode cho end customer, có button "Talk to human".
- Guardrails: PII mask, jailbreak detect, max 200 token output.
- HITL: confidence < 0.6 → human.
- Cost cap: $300/tháng (alert ở $250).

### Step 6 — Iterate (ongoing)

- Weekly eval review, monthly retrain reranker.
- Quarterly: đánh giá ROI, plan feature mới (multi-language, voice).

### Bài tập 15.1

Áp dụng quy trình 6 step cho 1 use case trong công ty bạn. Viết 1 trang mô tả mỗi bước.

---

## 3. Chi phí vận hành chuẩn

### 3.1 Cost breakdown điển hình

Cho app chatbot 100K request/tháng:

| Hạng mục | Cost/tháng |
|----------|-----------|
| LLM API (Claude Sonnet, ~5K token avg) | ~$1200 |
| Embedding (text-emb-3-small, daily refresh) | ~$50 |
| Re-ranking (Cohere) | ~$100 |
| Vector DB (Qdrant Cloud) | ~$80 |
| Cloud Run / Compute | ~$50 |
| Langfuse / Observability | ~$40 |
| Redis cache | ~$30 |
| **Total** | **~$1550/tháng** |

Cost / request ≈ **$0.0155**.

### 3.2 Bài học giảm chi phí

| Technique | Tiết kiệm |
|-----------|-----------|
| **Prompt caching** | -50% input cost (system prompt + tools dài) |
| **Model routing** | Haiku cho 70% câu đơn giản → -60% |
| **Semantic cache** | -25% với câu hỏi lặp |
| **Batch API** | -50% với offline job |
| **Cắt few-shot** thừa | -10-20% |
| **max_tokens** giới hạn | tránh spike |
| **Embedding cache** | embed query lặp 1 lần |

Áp dụng đủ: chi phí app trên có thể xuống **$500-700/tháng** (~50-60% saving).

### 3.3 Cost monitoring rules

```python
# Hard rules
PER_USER_DAILY_LIMIT  = 100     # request
PER_REQUEST_MAX_TOKEN = 4000    # input
PER_REQUEST_TIMEOUT   = 30      # second
PROJECT_DAILY_BUDGET  = 50      # USD

# Soft alerts
if hourly_cost > PROJECT_DAILY_BUDGET / 24 * 1.5:
    slack("⚠️ Hourly cost cao bất thường")
```

### 3.4 Khi nào nên fine-tune?

- Khi prompt > **3000 token** chỉ để dạy format/giọng điệu → fine-tune Haiku/mini = rẻ + nhanh hơn.
- Khi cần **chuyên môn hẹp** (luật VN, ngôn ngữ ít data).
- Khi đã có **5K-50K cặp** training data chất lượng.

Cost fine-tune Anthropic/OpenAI: ~$50-500 tuỳ size. Sau đó inference rẻ hơn.

### Bài tập 15.2

Tính cost cho app của bạn theo 3 kịch bản:
- 1K request/tháng
- 100K request/tháng
- 10M request/tháng

Ở scale nào nên fine-tune? Self-host? Đầu tư prompt caching?

---

## 4. Operations & On-call

### 4.1 Runbook mẫu

```markdown
## Incident: LLM latency p95 > 10s

### Triage
1. Check Langfuse → step nào chậm? (rewrite / retrieve / LLM / rerank)
2. Check Anthropic status page → provider có issue?
3. Check Cloud Run metrics → CPU/mem ok?

### Mitigation
- Provider issue → switch model fallback (set ENV `FALLBACK=true`)
- Retrieval slow → restart Qdrant pod
- Self issue → scale replica gấp đôi

### Resolve
- Root cause + permanent fix
- Update runbook

### Post-mortem
- Within 48h, blameless template
```

### 4.2 Quy mô team đề xuất

| Stage | Headcount | Vai trò |
|-------|-----------|---------|
| MVP | 1-2 | AI engineer (full-stack) |
| Beta | 3-4 | + Product, + ML/data |
| Production | 5-8 | + SRE, + QA, + security |
| Scale | 10-15 | + dedicated eval, dedicated prompt engineer |

### 4.3 Quy tắc văn hoá

- **Đo trước**, đoán sau.
- **Ship eval** trước khi ship feature.
- **Postmortem blameless**.
- **Prompt là code** — review, version, test.
- **Eval set là gold** — bảo vệ khỏi corruption.

---

## 5. Định hướng tiếp theo

### 5.1 Kỹ năng nên đào sâu

| Hướng | Resources |
|-------|-----------|
| **Agent SDK & MCP** | Anthropic docs, MCP spec |
| **Vision Language Models** | GPT-4V, Claude Vision, Gemini |
| **Reasoning models** | OpenAI o-series, DeepSeek, Claude extended thinking |
| **Fine-tuning** | HF transformers, PEFT, LoRA |
| **Inference optimization** | vLLM, TensorRT-LLM, llama.cpp |
| **AI Safety** | Anthropic interp, Redwood Research |

### 5.2 Career paths

```
                  AI / LLM Engineer
                /        |          \
        AI Eng     ML Research    AI Product
        (build)    (model R&D)    (product/PM)
           │             │              │
        +Agent      +Foundation     +Strategy
        +RAG        +Fine-tune      +Go-to-market
        +Ops        +Eval           +UX research
```

### 5.3 Cập nhật liên tục

- Papers: arxiv-sanity, [PapersWithCode](https://paperswithcode.com/)
- Newsletter: Latent Space, The Batch (Andrew Ng), Import AI
- Twitter/X: @_swarmw_ant, @karpathy, @sama, @AnthropicAI
- YouTube: Andrej Karpathy, Two Minute Papers, Yannic Kilcher
- Communities: Hugging Face, MLOps Community, [r/LocalLLaMA](https://www.reddit.com/r/LocalLLaMA)

### 5.4 Project idea để tiếp tục học

1. **Personal Knowledge Agent**: agent đọc Notion/Obsidian của bạn, trả lời câu hỏi.
2. **Code Review Agent**: agent review PR theo style guide team.
3. **Multi-modal**: bot đọc invoice ảnh → trích structured data.
4. **Voice Agent**: phone agent với TTS/STT.
5. **Local LLM**: deploy Llama 3.1 8B trên Mac/PC, hoàn toàn offline.

### Bài tập 15.3

Chọn 1 project, viết 1 trang plan (problem, solution, stack, milestones 4 tuần).

---

## 6. Checklist "AI app production-ready"

- [ ] PRD có Eval, Cost, Risk
- [ ] Golden set ≥ 50 case
- [ ] CI eval gate
- [ ] Guardrails: input + output filter
- [ ] Citation/fallback "I don't know"
- [ ] Observability: log + trace + metric
- [ ] Cost alert + budget cap
- [ ] Rate limit per user
- [ ] HITL flow rõ ràng
- [ ] Runbook + on-call rota
- [ ] HTTPS, secret manager, no PII in logs
- [ ] Disaster recovery / fallback model
- [ ] User feedback collection
- [ ] Monthly review + iterate

---

## 7. Tóm tắt 15 ngày

- AI/LLM hữu ích nhưng **không phải búa vạn năng** — chọn đúng bài toán.
- Build agent: **ReAct + tool calling + guardrails + eval**.
- RAG: chunking + hybrid + re-rank + contextual.
- Multi-agent chỉ khi cần. MCP là chuẩn cắm tool tương lai.
- LLMOps = **prompt-as-code + eval pipeline + monitoring + cost**.
- Cost optimize bằng cache, routing, batch.
- Production checklist 14 items.
- Học tiếp: Agent SDK, reasoning, fine-tune, multimodal.

## 8. Bài tập tổng kết

1. **Capstone**: Chọn 1 use case, build từ A→Z: PRD → prototype → eval → deploy → monitor. 4 tuần.
2. **Cost audit**: Tính chi phí app hiện tại của bạn. Áp dụng ≥ 3 cost optimization. Đo trước/sau.
3. **Eval report**: Viết report 2 trang về eval của hệ thống bạn: golden set, metric, fail analysis, action items.
4. **Open source contribution**: Fix 1 issue trong [Langfuse](https://github.com/langfuse/langfuse), [Ragas](https://github.com/explodinggradients/ragas), [LangGraph](https://github.com/langchain-ai/langgraph), hoặc viết 1 MCP server.
5. **Mentor**: Dạy lại 1 phần khoá này cho đồng nghiệp/bạn bè. "Teach to learn".
6. **Đọc tiếp**:
   - [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)
   - [Chip Huyen — AI Engineering book](https://www.oreilly.com/library/view/ai-engineering/9781098166298/)
   - [Eugene Yan — Patterns for Building LLM-based Systems](https://eugeneyan.com/writing/llm-patterns/)

---

> Hoàn thành 15 ngày! Day 16 sẽ là **chuyên sâu Ngày 1** — đào sâu vào AI/LLM Foundation từ góc nhìn nghiên cứu.
