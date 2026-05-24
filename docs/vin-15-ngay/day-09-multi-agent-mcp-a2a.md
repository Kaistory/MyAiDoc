---
title: Day 9 — Multi-Agent & MCP/A2A
sidebar_position: 9
---

# Day 9 — Multi-Agent & MCP/A2A

> **Mục tiêu**: Hiểu khi nào cần multi-agent, các pattern (supervisor, swarm, hierarchy), MCP (Model Context Protocol) và A2A (Agent-to-Agent communication).

## 1. Multi-Agent — Cấp độ cơ bản

**Single Agent** = 1 LLM với 1 loop + nhiều tool. Đủ cho 80% case.

**Multi-Agent** = nhiều LLM "instance" cùng làm việc, mỗi cái có vai trò / chuyên môn riêng.

```
Single:
   [User] ──► [Agent (làm hết)] ──► [User]

Multi:
   [User] ──► [Supervisor]
                  ├──► [Researcher]
                  ├──► [Coder]
                  └──► [Reviewer]
                  ▼
              [User]
```

### Khi nào cần multi-agent?

✅ Nên dùng:
- Bài toán có **vai trò rõ ràng** (research, code, review, plan).
- Cần **parallelize** (5 search task chạy song song).
- Mỗi agent có **prompt / tool / model khác nhau**.
- Workflow phức tạp, một agent dễ bối rối.

❌ Không nên:
- Bài toán đơn giản, single agent xử được.
- Sợ chi phí (multi-agent = nhân chi phí lên).
- Muốn debug dễ.

:::warning
Anthropic khuyến nghị: **bắt đầu với single agent**. Chỉ leo lên multi-agent khi single agent thực sự không gánh nổi. Multi-agent **đắt hơn 5-15 lần**.
:::

### Bài tập 9.1

Chọn 3 bài toán: (a) viết bài blog SEO, (b) dịch tài liệu kèm review, (c) chốt vé máy bay. Đề xuất: single hay multi? Vì sao?

---

## 2. Multi-Agent Patterns — Cấp độ trung cấp

### 2.1 Supervisor Pattern (Orchestrator-Worker)

Một agent **supervisor** điều phối, các agent con thực thi.

```
            ┌────────────────┐
            │  Supervisor    │
            │  - phân công   │
            │  - tổng hợp    │
            └───┬──────┬─────┘
                │      │
        ┌───────▼─┐  ┌─▼───────┐
        │ Worker 1│  │ Worker 2│
        └─────────┘  └─────────┘
```

Đây là pattern **phổ biến nhất**, dùng trong Claude Code, Claude Research, AutoGPT.

```python
# Pseudocode
def supervisor(user_request):
    plan = llm("Phân tích yêu cầu → tasks", user_request)
    results = []
    for task in plan.tasks:
        worker = pick_worker(task)
        results.append(worker.run(task))
    return llm("Tổng hợp kết quả", results)
```

### 2.2 Parallel / Fan-out

Khi N task **độc lập**, chạy song song.

```
                  [Supervisor]
                   │   │   │
            ┌──────┘   │   └──────┐
            ▼          ▼          ▼
         [Worker]  [Worker]   [Worker]   ←  chạy song song
            └──────┐   │   ┌──────┘
                   ▼   ▼   ▼
                  [Combiner]
```

Use case: research 5 chủ đề cùng lúc, search trên 4 source, transcribe nhiều file audio.

### 2.3 Hierarchy / Tree

Supervisor có **sub-supervisor**, mỗi sub-supervisor lại có worker. Cho task quá lớn.

```
       [Root Sup]
        │       │
   [Sub Sup]  [Sub Sup]
    │   │      │   │
   [W] [W]   [W] [W]
```

Anthropic dùng pattern này trong **Claude Research** mode.

### 2.4 Swarm / Peer-to-peer

Các agent cùng cấp, trao đổi với nhau. **CrewAI**, **OpenAI Swarm** đi theo style này.

```
   [Agent A] ◄──► [Agent B]
       ▲   ╲   ╱   ▲
       │    ╳      │
       ▼   ╱   ╲   ▼
   [Agent C] ◄──► [Agent D]
```

Khó kiểm soát hơn supervisor — dễ rơi vào tranh luận vô tận.

### 2.5 Pipeline / Chain

Output của agent N là input của N+1.

```
[Researcher] → [Writer] → [Editor] → [SEO Optimizer]
```

Đơn giản, dễ debug. Dùng khi quy trình **linear**.

### Bài tập 9.2

Vẽ kiến trúc multi-agent cho "phân tích báo cáo tài chính 200 trang":
- Pattern nào phù hợp?
- Liệt kê các vai trò agent.
- Mỗi agent có tool gì?

---

## 3. MCP — Model Context Protocol — Cấp độ nâng cao

**MCP** là chuẩn mở do **Anthropic** đề xuất (2024), cho phép LLM client (Claude Desktop, Claude Code, IDE...) kết nối với **MCP server** cung cấp tools / resources / prompts.

```
┌────────────────────┐     MCP      ┌────────────────────┐
│ Claude / IDE       │  ◄────────►  │ MCP Server         │
│ (MCP client)       │   JSON-RPC    │ (chrome, git, db,  │
│                    │               │  filesystem, ...)  │
└────────────────────┘               └────────────────────┘
```

### 3.1 Vì sao MCP quan trọng?

- **Plug-and-play**: viết tool 1 lần, dùng được ở nhiều client (Claude Desktop, Cursor, Continue...).
- **Tách biệt**: tool logic không nhúng vào prompt, không nhúng vào app.
- **An toàn**: client kiểm soát quyền, server chỉ expose theo schema.

### 3.2 Khái niệm chính của MCP

| Khái niệm | Mô tả |
|-----------|-------|
| **Tools** | Hàm có thể gọi (giống function calling) |
| **Resources** | Tài nguyên đọc-only (vd: file, DB record) |
| **Prompts** | Template prompt tái sử dụng |
| **Sampling** | Server xin client gọi LLM (advanced) |

### 3.3 Sample MCP server (Python)

```python
# my_mcp_server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Demo")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Cộng hai số nguyên."""
    return a + b

@mcp.tool()
def get_user_info(user_id: str) -> dict:
    """Lấy thông tin user từ DB nội bộ."""
    return {"id": user_id, "name": "Khải", "role": "engineer"}

@mcp.resource("file://notes/{filename}")
def read_note(filename: str) -> str:
    with open(f"./notes/{filename}") as f:
        return f.read()

if __name__ == "__main__":
    mcp.run()
```

Cấu hình Claude Desktop / Claude Code dùng server này:

```json
{
  "mcpServers": {
    "my-demo": {
      "command": "python",
      "args": ["/path/to/my_mcp_server.py"]
    }
  }
}
```

Sau khi cấu hình, Claude có thể tự gọi `add(2,3)` hoặc đọc file note.

### 3.4 Một số MCP server có sẵn

- **filesystem** — đọc/ghi file local.
- **git** — git operations.
- **github** — issues, PRs.
- **chrome-devtools / playwright** — automation browser.
- **postgres / sqlite** — query DB.
- **slack** — gửi tin nhắn.
- **memory** — knowledge graph cá nhân hoá.

Danh sách: https://github.com/modelcontextprotocol/servers

### Bài tập 9.3

1. Cài MCP filesystem server, cấu hình Claude Desktop/Code dùng nó. Cho Claude đọc & tóm tắt 1 file.
2. Viết MCP server riêng có 2 tool: `query_my_db`, `send_email`. Test với Claude Desktop.
3. Đọc spec MCP: https://spec.modelcontextprotocol.io/. Phân biệt **stdio transport** vs **HTTP transport**.

---

## 4. A2A — Agent-to-Agent Communication

**A2A** là khái niệm chung: agent giao tiếp với agent (không qua user).

Google công bố **A2A protocol** (2025) — chuẩn mở để agent từ vendor khác nhau "nói chuyện" với nhau (giống MCP nhưng giữa agent với agent).

```
[Agent A (vendor 1)] ◄── A2A protocol ──► [Agent B (vendor 2)]
       │                                          │
   Anthropic Claude                          OpenAI GPT
```

### 4.1 Use case

- Booking agent của bạn nói chuyện với agent của hãng máy bay.
- Internal HR agent giao tiếp với payroll agent.
- Multi-vendor enterprise: nhiều agent rời rạc nhưng cần phối hợp.

### 4.2 So sánh MCP vs A2A

| | MCP | A2A |
|--|------|------|
| **Mục đích** | Client ↔ Tool server | Agent ↔ Agent |
| **Pattern** | RPC-like | Task-based, message |
| **State** | Stateless (mỗi call độc lập) | Stateful (task có life-cycle) |
| **Authority** | Client gọi server | Hai bên peer |

:::info
MCP và A2A không thay thế nhau. Nhiều hệ thống dùng **cả hai**: agent gọi MCP để lấy tool, agent giao tiếp với agent khác qua A2A.
:::

### 4.3 Mini code A2A (mô phỏng)

```python
# Agent A có một skill, Agent B gọi qua HTTP
@app.post("/a2a/tasks")
def receive_task(task: TaskRequest):
    # Run agent logic
    result = my_agent.run(task.input)
    return {"task_id": task.id, "status": "completed", "output": result}
```

---

## 5. Best practices cho multi-agent

| Vấn đề | Best practice |
|--------|--------------|
| **Context blowup** | Mỗi agent có context riêng, summarize trước khi pass |
| **Cost explosion** | Track token mỗi worker, set budget |
| **Lỗi cascade** | Worker fail → retry, fallback, hoặc halt |
| **Race condition** | Khi worker chạy parallel, dùng async + locking |
| **Lost messages** | Persistent message queue (Redis, SQS) |
| **Debug khó** | Log từng turn, có UI hiển thị graph |
| **Loop vô tận** | `max_depth`, `max_turns` |

## 6. Frameworks

| Framework | Đặc điểm |
|-----------|----------|
| **LangGraph** | Build dạng graph, kiểm soát state tốt |
| **CrewAI** | Role-based, dễ học |
| **AutoGen** | Conversation-based, Microsoft |
| **OpenAI Swarm** | Lightweight, peer-to-peer |
| **Claude Agent SDK** | Anthropic, native MCP support |
| **DSPy** | Compile prompt tự động |

---

## 7. Tóm tắt

- **Single agent** trước. Chỉ multi-agent khi cần.
- Pattern phổ biến: **Supervisor**, **Parallel fan-out**, **Pipeline**, **Hierarchy**.
- **MCP** = chuẩn cắm tool vào client. Viết server 1 lần dùng ở nhiều nơi.
- **A2A** = chuẩn agent ↔ agent (Google, 2025).
- Multi-agent đắt: track cost, set budget, có stop condition.
- Debug bằng log chi tiết + UI graph.

## 8. Bài tập tổng hợp

1. **Build supervisor**: Viết hệ thống 3 agent: Planner → Researcher → Writer. Yêu cầu: "Viết bài blog 500 từ về MCP". So sánh chất lượng vs single agent.
2. **MCP server**: Viết MCP server expose API của 1 service nội bộ (vd: Jira). Test gọi qua Claude.
3. **Đọc**:
   - [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)
   - [MCP spec](https://spec.modelcontextprotocol.io/)
   - [Google A2A blog](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
4. **Phân tích**: Claude Code (CLI bạn đang dùng) là single hay multi-agent? Bằng chứng?
5. **Cost analysis**: Đo token usage của single vs multi-agent cho cùng nhiệm vụ. Chênh bao nhiêu lần?

---

> Hết Day 9. Mai sẽ học **Data Pipeline** và **Data Observability** cho AI system.
