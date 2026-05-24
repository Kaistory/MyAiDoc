---
title: Day 5 — AI Product Thinking & Requirements
sidebar_position: 5
---

# Day 5 — AI Product Thinking & Requirements

> **Mục tiêu**: Viết được PRD cho sản phẩm AI, đánh giá Risk Assessment, hiểu khác biệt giữa product truyền thống và product AI.

## 1. AI Product khác Product thường ra sao? — Cấp độ cơ bản

| Product thường | Product AI |
|----------------|-----------|
| Deterministic — input X luôn cho Y | Probabilistic — input X có thể cho Y, Y', Y'' |
| Bug có thể fix triệt để | "Bug" đôi khi là phân phối — chỉ giảm được tỷ lệ |
| Test cases rõ | Cần **eval set**, accuracy threshold |
| UX: button, form | UX: chat, suggest, edit, undo |
| Cost: server, băng thông | Cost: **token** (input + output, scale theo user) |
| Quality stable | Quality **drift** theo data, model update |

### Hệ quả thực tế

- **UX**: phải show *confidence*, cho phép user **edit / regenerate / undo**.
- **Reliability**: cần **fallback** (rule-based) khi LLM fail.
- **Cost**: dự đoán cost theo user growth — có thể tăng đột biến.
- **Eval**: phải có metric, không thể "tin cảm tính".

### Bài tập 5.1

Chọn 1 feature AI bạn từng dùng (vd: GitHub Copilot suggestion, Notion AI summarize). Liệt kê 3 cách UX xử lý "AI có thể sai" (confidence, undo, edit, fallback...).

---

## 2. PRD cho AI Product — Cấp độ trung cấp

**PRD** (Product Requirements Document) cho AI nên có thêm các mục đặc thù:

### Template PRD cho AI

```markdown
# PRD: [Tên feature]

## 1. Vấn đề & User
- User là ai? Pain point gì? Hiện đang giải quyết ra sao?

## 2. Success Metrics
- Bắc baccu (north star): vd CSAT, time saved, conversion.
- Proxy: accuracy, recall, latency, cost/request.

## 3. Solution
- Cách giải quyết. Có những lựa chọn nào (rule, ML, LLM)?
- Vì sao chọn AI?

## 4. Model & Data
- Model nào? Vì sao?
- Dữ liệu input: format, kích thước.
- Có cần fine-tune / RAG không?

## 5. UX khi AI sai
- Tỷ lệ sai chấp nhận được: vd < 5%.
- Cách user phát hiện & sửa.
- Fallback khi LLM down / slow.

## 6. Guardrails
- PII (thông tin cá nhân) xử lý sao?
- Content moderation?
- Rate limit?
- Hành vi bị cấm (vd: bot không nói chính trị).

## 7. Eval Plan
- Bộ test (golden set): bao nhiêu mẫu, ai làm?
- Metric: accuracy, BLEU, ROUGE, human eval?
- Ngưỡng pass / fail?
- Cách re-eval khi đổi model.

## 8. Cost
- Token/request × số request/ngày × giá → cost/tháng.
- Cost ceiling, monitor cảnh báo.

## 9. Risk Assessment
(Xem mục 3 dưới)

## 10. Rollout Plan
- Internal alpha → beta % user → 100%.
- A/B test với baseline.
- Kill switch (tắt nhanh nếu lỗi).

## 11. Timeline & Resource
```

### Bài tập 5.2

Lấy bài toán từ Day 2 (bài tập 2.3), viết PRD đầy đủ theo template trên (~2 trang).

---

## 3. Risk Assessment — Cấp độ nâng cao

AI mở ra các loại risk **mới**. Phải nhận diện và mitigate.

### 3.1 Ma trận risk

| Loại risk | Ví dụ | Mitigation |
|-----------|-------|------------|
| **Hallucination** | LLM bịa thông tin sai | RAG, citation, "không biết" mặc định |
| **PII leak** | LLM nhả số CMND/CCCD | Mask trước/sau, DLP |
| **Prompt injection** | User chèn lệnh "ignore previous instructions" | Validate input, system prompt cứng |
| **Jailbreak** | User bypass guardrail | Multi-layer filter, classifier riêng |
| **Bias** | Model thiên lệch giới tính/sắc tộc | Eval set có bias test, fine-tune |
| **Copyright** | Output copy từ training data | Citation, fair use review |
| **Cost runaway** | Bug khiến gọi API vô hạn | Rate limit, budget alert |
| **Model drift** | Provider update model → behavior khác | Pin version, regression test |
| **Latency spike** | LLM chậm bất thường | Timeout, async, fallback model |
| **Vendor lock-in** | Phụ thuộc 1 provider | Abstraction layer, multi-provider |
| **Compliance** | GDPR, HIPAA, NIS2 | Legal review, data residency |
| **Liability** | AI khuyên sai → user kiện | T&C rõ ràng, human-in-the-loop |

### 3.2 Risk Scoring

Cho mỗi risk, tính: **Likelihood × Impact**.

```
       Impact: Thấp │ Trung │ Cao  │ Nghiêm trọng
Likelihood ────────┼───────┼──────┼──────────────
   Hiếm           │  1    │  2   │  3   │   4
   Thi thoảng     │  2    │  4   │  6   │   8
   Thường         │  3    │  6   │  9   │  12
   Liên tục       │  4    │  8   │ 12   │  16
```

Risk ≥ 8 → bắt buộc có mitigation rõ ràng + owner.

### 3.3 Threat Model cho LLM app

Đi qua **STRIDE-LM** (mở rộng từ STRIDE cho LLM):

- **S**poofing — giả mạo identity (vd: chèn role giả vào prompt)
- **T**ampering — sửa prompt/output
- **R**epudiation — phủ nhận hành động
- **I**nfo disclosure — lộ system prompt, lộ data người khác
- **D**enial of Service — prompt cố tình gây loop, tốn token
- **E**levation of privilege — bypass guardrail
- **L**LM-specific: jailbreak, prompt injection, training data poisoning

### Bài tập 5.3

Cho **chatbot tư vấn pháp luật** dùng LLM + RAG:
1. Liệt kê **6 risk** lớn nhất.
2. Cho điểm Likelihood × Impact.
3. Đề xuất mitigation cho 3 risk cao nhất.

---

## 4. Sample Risk Register

```markdown
## R-001: Hallucination về điều luật
- Likelihood: Thường
- Impact: Nghiêm trọng (user dùng sai luật)
- Score: 12
- Owner: AI Eng Lead
- Mitigation:
  1. RAG với corpus luật chính thức, citation bắt buộc.
  2. Disclaimer trên mọi câu trả lời.
  3. HITL: chuyển luật sư khi câu hỏi phức tạp.
  4. Eval set 200 câu hỏi luật, accuracy > 95%.

## R-002: PII leak từ document upload
- Likelihood: Thường
- Impact: Cao
- Score: 9
- Owner: Security
- Mitigation:
  1. DLP scan trước khi index vào vector DB.
  2. Encrypt at rest, scoped access.
  3. Auto-delete sau 90 ngày.
```

---

## 5. UX patterns cho AI Product

| Pattern | Mô tả | Ví dụ |
|---------|-------|-------|
| **Suggest, don't decide** | Đề xuất, user chọn | Copilot inline suggestion |
| **Show confidence** | Hiển thị độ tin cậy | "AI 80% chắc" |
| **Easy undo / regenerate** | Sai thì hoàn tác nhanh | ChatGPT regenerate |
| **Edit, don't replace** | Cho user sửa output | Notion AI |
| **Transparency** | Cite source, show reasoning | Perplexity citation |
| **Progressive disclosure** | Hiện kết quả từng phần | Streaming response |
| **Calibrated expectation** | Disclaimer rõ ràng | "AI có thể sai, hãy kiểm tra" |

---

## 6. Tóm tắt

- AI product **xác suất**, khác hẳn product truyền thống — UX phải accomodate sai số.
- PRD AI có thêm: **Eval, Cost, Guardrails, Risk, Rollout**.
- **Risk register** rõ ràng, owner cụ thể, mitigation đo được.
- Áp dụng UX patterns: **suggest, undo, edit, cite**.
- Đừng quên **kill switch** & **cost cap**.

## 7. Bài tập tổng hợp

1. **PRD hoàn chỉnh**: Viết PRD ~4 trang cho 1 trong các ý tưởng:
   - Bot tóm tắt email
   - Trợ lý viết caption mạng xã hội
   - Search nội bộ công ty
2. **Risk register**: Cho PRD trên, liệt kê ≥ 8 risk, scoring, mitigation, owner.
3. **UX mockup**: Vẽ wireframe 3 màn hình cho product trên, chú thích chỗ nào "AI có thể sai" và xử lý ra sao.
4. **Đọc**:
   - [Google PAIR Guidebook](https://pair.withgoogle.com/guidebook/)
   - [Microsoft Responsible AI Standard](https://www.microsoft.com/en-us/ai/responsible-ai)
   - [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
5. **Case study**: Tìm 1 case AI gây thiệt hại (Air Canada chatbot, Samsung leak source code...). Nó vi phạm risk nào?

---

> Hết Day 5. Mai sẽ **làm tay** một prototype hackathon trong 1 ngày!
