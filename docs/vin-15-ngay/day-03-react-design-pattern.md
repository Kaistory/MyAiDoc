---
title: Day 3 — Design Pattern ReAct
sidebar_position: 3
---

# Day 3 — Design Pattern ReAct (Reasoning + Acting)

> **Mục tiêu**: Hiểu kiến trúc Agent, ReAct là gì, Agent Loop hoạt động ra sao qua chu trình Thought → Action → Observation. Tự code một ReAct agent mini từ con số 0.

## 1. Agent là gì? — Cấp độ cơ bản

Một **AI Agent** = LLM + khả năng **gọi tool** + **vòng lặp tự ra quyết định**.

Khác với một LLM chat đơn thuần (chỉ trả lời 1 câu):

```
Chat LLM:     [User question] ──► [LLM]  ──► [Answer]

Agent:        [User goal] ──► [LLM] ──► [Tool?] ──► [Observation]
                              ▲                          │
                              └──────────────────────────┘
                                  (lặp đến khi xong)
```

### Ví dụ trực quan

Yêu cầu: *"Thời tiết Hà Nội hôm nay thế nào, và có nên đi picnic?"*

- LLM thường: "Tôi không có dữ liệu thời tiết realtime" 🙅
- Agent:
  1. **Thought**: cần gọi API thời tiết.
  2. **Action**: `get_weather(city="Hà Nội")`
  3. **Observation**: `{"temp": 28, "rain": 70%}`
  4. **Thought**: mưa 70% — không nên picnic.
  5. **Answer**: "Hà Nội 28°C, khả năng mưa 70%. Hôm nay không nên picnic, đợi cuối tuần."

### Bài tập 3.1

Liệt kê 3 ví dụ Agent bạn từng dùng (gợi ý: Claude Code, Cursor, ChatGPT với "browse the web"). Mỗi agent gọi tool gì?

---

## 2. ReAct Pattern — Cấp độ trung cấp

**ReAct** = **Rea**soning + **Act**ing. Paper gốc: [ReAct (Yao et al., 2022)](https://arxiv.org/abs/2210.03629).

Ý tưởng: LLM **luân phiên** giữa suy nghĩ (Thought) và hành động (Action), quan sát kết quả (Observation) rồi suy nghĩ tiếp.

### 2.1 Chu trình Thought-Action-Observation

```
┌─────────────────────────────────────────────────┐
│                  Agent Loop                      │
│                                                  │
│   ┌──────────┐                                  │
│   │ Thought  │ "Tôi cần tìm dân số Hà Nội"     │
│   └────┬─────┘                                  │
│        │                                         │
│        ▼                                         │
│   ┌──────────┐                                  │
│   │ Action   │ search("dân số Hà Nội 2025")    │
│   └────┬─────┘                                  │
│        │                                         │
│        ▼                                         │
│   ┌──────────────┐                              │
│   │ Observation  │ "8.5 triệu"                  │
│   └────┬─────────┘                              │
│        │                                         │
│        ▼                                         │
│   ┌──────────┐                                  │
│   │ Thought  │ "Đủ thông tin, trả lời thôi"    │
│   └────┬─────┘                                  │
│        │                                         │
│        ▼                                         │
│   ┌──────────┐                                  │
│   │ Final    │ "Dân số Hà Nội năm 2025: 8.5tr" │
│   └──────────┘                                  │
└─────────────────────────────────────────────────┘
```

### 2.2 Prompt template kinh điển

```
Bạn là một AI agent. Hãy giải bài toán bằng cách dùng vòng lặp:
Thought: <suy nghĩ của bạn>
Action: <tên tool>(<tham số>)
Observation: <kết quả từ tool>
... (lặp lại)
Thought: Tôi đã đủ thông tin.
Final Answer: <câu trả lời cuối>

Các tool có sẵn:
- search(query): tìm trên web.
- calculator(expression): tính toán.

Câu hỏi: {question}
```

### 2.3 Code minh hoạ — ReAct mini bằng Python thuần

```python
import re
from anthropic import Anthropic

client = Anthropic()

TOOLS = {
    "calculator": lambda expr: str(eval(expr)),
    "search": lambda q: f"[fake search result for: {q}]",
}

SYSTEM = """Bạn là ReAct agent. Mỗi lượt phải in:
Thought: ...
Action: tool_name(args)
hoặc khi xong:
Final Answer: ...

Tool có sẵn: calculator(expr), search(query).
"""

def run(question: str, max_steps: int = 6):
    history = f"Câu hỏi: {question}\n"
    for step in range(max_steps):
        resp = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            system=SYSTEM,
            messages=[{"role": "user", "content": history}],
            stop_sequences=["Observation:"],
        )
        text = resp.content[0].text
        history += text

        if "Final Answer:" in text:
            print(history)
            return text.split("Final Answer:")[-1].strip()

        m = re.search(r"Action:\s*(\w+)\((.*)\)", text)
        if not m:
            break
        tool, args = m.group(1), m.group(2).strip(' "\'')
        obs = TOOLS[tool](args)
        history += f"\nObservation: {obs}\n"

    return "Hết bước cho phép."

print(run("Tính 2024 * 365 cộng với dân số Hà Nội"))
```

### Bài tập 3.2

1. Chạy code trên, thay `search` bằng tool thật (vd: gọi DuckDuckGo API).
2. Thêm tool `wikipedia(topic)` trả về tóm tắt từ Wikipedia.
3. Thử câu hỏi đa bước: *"Tìm năm sinh của Albert Einstein, lấy bình phương rồi cộng 100."* Agent đi qua bao nhiêu Thought-Action?

---

## 3. Agent Loop nâng cao — Cấp độ nâng cao

### 3.1 Vòng lặp đầy đủ

```
loop:
  1. Build prompt = system + history (messages, tool results)
  2. Call LLM
  3. Parse response:
       - Plain text  → trả về cho user
       - Tool calls  → thực thi từng tool
  4. Append tool results vào history
  5. Continue loop (back to 1)
Stop conditions:
  - LLM không gọi tool nữa
  - Đạt max_iterations
  - Lỗi không recoverable
  - User dừng
```

### 3.2 Dùng **native tool calling** thay vì regex

API hiện đại (Anthropic, OpenAI) hỗ trợ **tool calling structured**, không cần parse text thủ công.

```python
from anthropic import Anthropic
client = Anthropic()

tools = [{
    "name": "calculator",
    "description": "Tính một biểu thức toán học.",
    "input_schema": {
        "type": "object",
        "properties": {"expression": {"type": "string"}},
        "required": ["expression"],
    },
}]

messages = [{"role": "user", "content": "Tính (123 + 456) * 7"}]

while True:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        tools=tools,
        messages=messages,
    )
    messages.append({"role": "assistant", "content": resp.content})

    if resp.stop_reason != "tool_use":
        print(resp.content[-1].text)
        break

    tool_results = []
    for block in resp.content:
        if block.type == "tool_use":
            result = str(eval(block.input["expression"]))
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result,
            })
    messages.append({"role": "user", "content": tool_results})
```

### 3.3 Các biến thể quan trọng

| Pattern | Mô tả | Khi nào dùng |
|---------|-------|--------------|
| **ReAct** | Thought → Action → Obs | Default, dễ debug |
| **Plan-and-Execute** | Lập kế hoạch trước, thực thi sau | Bài toán nhiều bước, ít rẽ nhánh |
| **Reflexion** | Phản tỉnh sau mỗi lần thất bại | Khi cần học từ lỗi |
| **Chain-of-Thought** | Chỉ suy luận, không tool | Bài toán logic thuần |
| **Tree-of-Thoughts** | Mở nhiều nhánh, đánh giá, chọn nhánh tốt nhất | Bài toán cần khám phá |

### 3.4 Best practices

> 💡 **Mẹo — Production-grade ReAct**
>
> - **Giới hạn vòng lặp**: `max_iterations` (10-20) — tránh chạy vô hạn.
> - **Timeout cho mỗi tool**: tránh tool treo.
> - **Truncate history**: khi context dài, summarize đoạn cũ.
> - **Tool descriptions phải rõ**: LLM gọi đúng tool, đúng tham số.
> - **Validate tool input**: dùng JSON Schema, từ chối input xấu.
> - **Log tất cả**: thought, action, observation — để debug.
> - **Streaming**: hiển thị thought ra UI để user thấy agent đang làm gì.

### Bài tập 3.3

1. Refactor code phần 2.3 (regex) sang **native tool calling** (phần 3.2).
2. Thêm `max_iterations=10`. Tạo câu hỏi cố tình lặp vô hạn — agent dừng đúng không?
3. Thêm cơ chế **summarize** khi history > 5000 token.

---

## 4. Anti-pattern khi xây Agent

> ⚠️ **Cảnh báo — Sai lầm**
>
> 1. **Quá nhiều tool**: LLM bối rối, chọn sai. Bắt đầu với 3-5 tool.
> 2. **Tool description mơ hồ**: "Search something" — LLM gọi bừa.
> 3. **Không có stop condition**: agent lặp mãi → cháy quota.
> 4. **Tin tưởng tuyệt đối**: agent có thể sai, **đừng cho tool nguy hiểm** (xoá file, gửi email) mà không xác nhận.
> 5. **Không log**: lỗi xảy ra mà không biết step nào sai.

## 5. So sánh với các framework

| Framework | Đặc điểm |
|-----------|----------|
| **LangChain** | Cũ, nhiều tính năng, đôi khi over-engineered |
| **LangGraph** | Build agent dạng graph, kiểm soát flow rõ |
| **LlamaIndex** | Mạnh ở RAG, có Agent support |
| **AutoGen (Microsoft)** | Multi-agent, conversation-based |
| **CrewAI** | Multi-agent với role |
| **Anthropic Agent SDK** | Mới, gọn, hợp với Claude |
| **Tự code** | Nhẹ nhất, ít magic — recommend cho người mới |

> 💡 **Mẹo — Khuyến nghị**
>
> **Bắt đầu**: tự code 50 dòng Python, hiểu loop.
> **Production**: chuyển sang LangGraph hoặc Agent SDK khi cần.

---

## 6. Tóm tắt

- **Agent = LLM + Tool + Loop**.
- **ReAct** xen kẽ **Thought / Action / Observation**.
- Dùng **native tool calling** thay vì regex (an toàn hơn).
- Đặt **stop conditions**, **timeout**, **logging**.
- Bắt đầu với **3-5 tool**, mô tả **rõ ràng**.
- Khi đủ tự tin, leo lên **Plan-and-Execute, Reflexion**.

## 7. Bài tập tổng hợp

1. **Build mini agent**: Viết ReAct agent có 3 tool: `calculator`, `today_date()`, `read_file(path)`. Hỏi: *"File `notes.txt` chứa bao nhiêu từ?"*
2. **So sánh**: Code cùng bài toán bằng 2 cách: regex parsing vs native tool calling. Đo: số dòng, độ ổn định, khả năng debug.
3. **Đọc paper**: [ReAct paper](https://arxiv.org/abs/2210.03629) — đọc phần 3 (Method), tóm tắt sự khác biệt giữa ReAct, Act-only, CoT-only.
4. **Reflexion**: Đọc [Reflexion paper (2023)](https://arxiv.org/abs/2303.11366). Khi nào nó vượt ReAct?
5. **Real-world**: Tìm hiểu cách Claude Code (CLI bạn đang dùng) implement agent loop. Nó dùng tool gì, dừng khi nào?

---

> Hết Day 3. Mai sẽ deep-dive **Prompt Engineering** và **Tool Calling** thật chuẩn.
