---
title: Day 4 — Prompt Engineering & Tool Calling
sidebar_position: 4
---

# Day 4 — Prompt Engineering & Tool Calling

> **Mục tiêu**: Biết viết prompt chuẩn (system, user, assistant), nắm chain-of-thought, few-shot, structured output, và tool/function calling đúng chuẩn API.

## 1. Cấu trúc prompt — Cấp độ cơ bản

Một message gửi tới LLM thường có 3 role:

```
┌──────────────────────────────────────┐
│ system    │ Vai trò, ràng buộc, format │
├──────────────────────────────────────┤
│ user      │ Câu hỏi / yêu cầu          │
├──────────────────────────────────────┤
│ assistant │ Phản hồi (history)         │
└──────────────────────────────────────┘
```

### Ví dụ với Anthropic API

```python
from anthropic import Anthropic
client = Anthropic()

resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="Bạn là trợ lý lập trình Python, trả lời ngắn gọn, có code.",
    messages=[
        {"role": "user", "content": "Cách đọc file CSV trong Python?"},
        {"role": "assistant", "content": "Dùng `pandas.read_csv()`..."},
        {"role": "user", "content": "Nếu file quá lớn?"},
    ],
)
print(resp.content[0].text)
```

:::info System prompt
**System** là chỗ "lập trình hành vi" của LLM. Đặt: vai trò, giọng điệu, định dạng đầu ra, điều cấm.
:::

### Bài tập 4.1

Viết system prompt cho 3 chatbot khác nhau: (a) trợ lý du lịch Việt Nam, (b) reviewer code Go, (c) hướng dẫn viên thiền. So sánh độ dài và mức ràng buộc.

---

## 2. Kỹ thuật prompt phổ biến — Cấp độ trung cấp

### 2.1 Zero-shot vs Few-shot

**Zero-shot**: chỉ mô tả, không ví dụ.

```
Phân loại review sau là tích cực/tiêu cực/trung tính:
"Sản phẩm dùng ổn, nhưng giao hàng chậm."
```

**Few-shot**: cho vài ví dụ trước.

```
Phân loại review:

Review: "Tuyệt vời, giao nhanh!"        → Tích cực
Review: "Tệ, hỏng ngay sau 1 tuần."      → Tiêu cực
Review: "Không quá tệ, không quá tốt."   → Trung tính

Review: "Sản phẩm dùng ổn, nhưng giao hàng chậm." →
```

Few-shot thường **chính xác hơn nhiều** với task khó, đặc biệt khi format đầu ra phức tạp.

### 2.2 Chain-of-Thought (CoT)

Bắt LLM "nghĩ ra giấy" trước khi trả lời cuối:

```
Q: Lan có 5 quả táo, cho An 2 quả, mẹ cho thêm 3. Lan còn bao nhiêu?

A: Hãy giải từng bước.
- Ban đầu Lan có 5 quả.
- Cho An 2 quả → 5 - 2 = 3.
- Mẹ cho 3 quả → 3 + 3 = 6.
Đáp số: 6 quả.
```

Đặc biệt hiệu quả với toán, logic, code. Nhiều model hiện đại (Claude, o-series) có **extended thinking** mode — tự nghĩ trong block riêng.

### 2.3 Structured Output (JSON, XML, schema)

Bắt LLM trả về dạng có cấu trúc để code dễ parse.

#### Dùng JSON mode (OpenAI)

```python
resp = client.chat.completions.create(
    model="gpt-4.1",
    response_format={"type": "json_schema", "json_schema": {
        "name": "review_classification",
        "schema": {
            "type": "object",
            "properties": {
                "sentiment": {"enum": ["positive", "negative", "neutral"]},
                "confidence": {"type": "number"},
                "reasons": {"type": "array", "items": {"type": "string"}},
            },
            "required": ["sentiment", "confidence", "reasons"],
        }
    }},
    messages=[...],
)
```

#### Dùng XML tag (Claude)

Claude rất giỏi xử lý XML. Pattern phổ biến:

```
<review>Sản phẩm tệ, không nên mua</review>

Hãy trả về:
<analysis>
  <sentiment>...</sentiment>
  <reasons>...</reasons>
</analysis>
```

### 2.4 Các kỹ thuật nâng cao

| Kỹ thuật | Mô tả |
|----------|-------|
| **Role prompting** | "Bạn là chuyên gia X..." |
| **Self-consistency** | Sinh nhiều answer, lấy majority vote |
| **CoT** | "Hãy nghĩ từng bước" |
| **Tree-of-Thoughts** | Mở nhiều nhánh suy luận |
| **Prompt chaining** | Chia bài toán thành nhiều prompt nhỏ |
| **Self-critique** | Bắt LLM tự review câu trả lời |
| **Constitutional AI** | Đặt nguyên tắc, bắt LLM tự tuân thủ |

### Bài tập 4.2

1. Viết 1 prompt zero-shot và 1 prompt few-shot cho task: trích xuất `{tên, email, sđt}` từ chữ ký email. Test 5 chữ ký, so sánh accuracy.
2. Thêm CoT vào prompt giải toán lớp 5 — có cải thiện không?
3. Viết prompt yêu cầu trả về JSON Schema cụ thể. Đo: tỷ lệ trả về JSON hợp lệ.

---

## 3. Tool Calling chuẩn — Cấp độ nâng cao

### 3.1 Tool calling là gì?

LLM thông thường chỉ trả văn bản. **Tool calling** cho phép LLM "yêu cầu" code chạy một hàm, rồi đọc kết quả để trả lời.

```
   User: "Thời tiết Hà Nội?"
        │
        ▼
   LLM: <tool_use name="get_weather" input={city:"HN"}/>
        │
        ▼
   Code: gọi API thời tiết → {"temp":28}
        │
        ▼
   LLM: "Hà Nội đang 28°C."
```

### 3.2 Định nghĩa tool

JSON Schema mô tả tên, mô tả, tham số. Schema **càng rõ** thì LLM gọi **càng đúng**.

```json
{
  "name": "get_weather",
  "description": "Lấy thời tiết hiện tại của một thành phố. Trả về nhiệt độ (°C), độ ẩm (%), tình trạng (mưa/nắng/mây).",
  "input_schema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "Tên thành phố tiếng Anh, vd: 'Hanoi', 'Ho Chi Minh City'."
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "default": "celsius"
      }
    },
    "required": ["city"]
  }
}
```

### 3.3 Code đầy đủ một vòng tool calling

```python
from anthropic import Anthropic
import json

client = Anthropic()

def get_weather(city: str, unit: str = "celsius"):
    return {"city": city, "temp": 28, "unit": unit, "condition": "mây"}

TOOLS = [{
    "name": "get_weather",
    "description": "Lấy thời tiết hiện tại của một thành phố.",
    "input_schema": {
        "type": "object",
        "properties": {
            "city": {"type": "string"},
            "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
        },
        "required": ["city"],
    },
}]

messages = [{"role": "user", "content": "Thời tiết Hà Nội và Sài Gòn?"}]

while True:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        tools=TOOLS,
        messages=messages,
    )
    messages.append({"role": "assistant", "content": resp.content})

    if resp.stop_reason != "tool_use":
        for block in resp.content:
            if hasattr(block, "text"):
                print(block.text)
        break

    tool_results = []
    for block in resp.content:
        if block.type == "tool_use":
            if block.name == "get_weather":
                result = get_weather(**block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": json.dumps(result),
                })
    messages.append({"role": "user", "content": tool_results})
```

:::tip Parallel tool calls
Một số model có thể gọi **nhiều tool song song** trong 1 lượt (vd: 2 thành phố). Đảm bảo code xử lý nhiều `tool_use` block trong cùng response.
:::

### 3.4 Best practices

| Vấn đề | Cách xử lý |
|--------|-----------|
| **Tool description không rõ** | Viết như docstring cho dev mới, có ví dụ |
| **Tham số sai kiểu** | Dùng JSON Schema strict, validate trước khi gọi |
| **Tool fail** | Trả error message rõ ràng vào tool_result |
| **Tool nguy hiểm** | Yêu cầu user xác nhận trước khi thực thi |
| **Quá nhiều tool** | Group thành namespace, hoặc filter theo intent |
| **Tool chậm** | Async, parallel, có timeout |

### 3.5 Caching prompt (giảm chi phí)

API Claude/OpenAI hỗ trợ **prompt caching**: system prompt dài, tool definitions sẽ được cache 5 phút, lượt sau chỉ trả phí ~10% so với input thường.

```python
resp = client.messages.create(
    model="claude-sonnet-4-6",
    system=[
        {
            "type": "text",
            "text": "Bạn là agent rất dài...",
            "cache_control": {"type": "ephemeral"},
        }
    ],
    tools=TOOLS,  # tools cũng cache
    messages=messages,
)
```

:::info Khi nào cache có lợi?
- System prompt **> 1024 token**.
- Lượt gọi **lặp đi lặp lại** trong < 5 phút.
- Multi-turn chat với cùng system.
:::

### Bài tập 4.3

1. Viết agent có **3 tool**: `get_weather`, `search_flight`, `book_flight`. Yêu cầu xác nhận trước khi `book_flight`.
2. Thử **parallel tool calls**: hỏi thời tiết 5 thành phố cùng lúc.
3. Thêm prompt caching vào system prompt 2000 token. So sánh chi phí 10 lượt gọi.

---

## 4. Anti-pattern khi prompt

:::warning Sai lầm
1. **Prompt quá dài, mơ hồ**: "Hãy thông minh và trả lời tốt nhất có thể" — vô nghĩa.
2. **Trộn nhiều task** trong 1 prompt mà không cấu trúc.
3. **Không validate output**: LLM trả JSON sai cú pháp → app crash.
4. **Không có guardrail**: LLM lộ thông tin, làm việc cấm.
5. **Hardcode prompt trong code**: khó test, khó update.
:::

## 5. Prompt template framework

**Khuyến nghị tổ chức**: tách prompt ra file riêng, version control, có test.

```
prompts/
  classify_review.v1.md
  classify_review.v2.md   # version mới, A/B test
  extract_invoice.v1.md
```

Tools: [PromptLayer](https://promptlayer.com/), [LangSmith](https://smith.langchain.com/), [Helicone](https://www.helicone.ai/).

---

## 6. Tóm tắt

- 3 role: **system / user / assistant**. System đặt vai trò.
- Few-shot > Zero-shot cho task khó.
- **CoT** giúp logic, toán, code.
- **Structured output** (JSON schema, XML) cho code dễ parse.
- **Tool calling**: định nghĩa rõ ràng, validate, có timeout, log đầy đủ.
- **Prompt caching** giảm chi phí > 80% trong multi-turn.
- Tổ chức prompt như **code asset**: version, test, A/B.

## 7. Bài tập tổng hợp

1. **Refactor**: Tìm 1 prompt cũ trong project của bạn, viết lại thành: system + few-shot + structured output. Đo lại chất lượng.
2. **Tool**: Viết tool `query_sql(database, query)` cho LLM. Quan trọng — chặn các câu DROP, DELETE. Mô tả tool sao cho LLM không cố gọi sai.
3. **A/B prompt**: Soạn 2 phiên bản system prompt cho cùng task. Chạy 50 input, scoring bằng LLM khác. Bản nào tốt hơn?
4. **Đọc**: [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/claude/docs/prompt-engineering), [OpenAI Prompting](https://platform.openai.com/docs/guides/prompt-engineering). Tóm tắt 5 nguyên tắc lớn.
5. **Đo**: Cache hit rate của project bạn — bao nhiêu %? Có cách nào tăng không?

---

> Hết Day 4. Mai sẽ học **AI Product Thinking** — cách viết PRD và đánh giá risk.
