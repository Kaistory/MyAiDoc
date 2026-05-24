---
title: Day 6 — AI Product Prototype & Demo (Hackathon)
sidebar_position: 6
---

# Day 6 — AI Product Prototype & Demo (Hackathon)

> **Mục tiêu**: Build được một prototype AI từ 0 → demo trong **1 ngày**. Hiểu tech stack tối thiểu, kỹ năng demo, và những sai lầm thường gặp ở hackathon.

## 1. Mindset hackathon — Cấp độ cơ bản

Hackathon **không phải** xây production-grade. Là về:

- **Idea rõ ràng** (1 câu nói lên problem + solution).
- **Demo chạy được** trên màn hình thật, không phải slide.
- **Story hấp dẫn** — vì sao quan trọng.

### Quy tắc "1-day prototype"

```
0-2h:  Ý tưởng + scope (đừng tham!)
2-4h:  Wireframe + tech stack
4-9h:  Build core feature (chỉ 1!)
9-10h: Fake data, polish UI
10-11h: Tập demo, quay video backup
11-12h: Demo / pitch
```

> ⚠️ **Cảnh báo — Bẫy thường gặp**
>
> - **Scope creep**: thêm tính năng mỗi 30 phút → 5h sau không cái nào chạy.
> - **Auth/database từ đầu**: tốn 3h cho thứ không demo được.
> - **UI quá đẹp**: tốn thời gian, không add giá trị demo.
> - **Cố show off**: thêm RAG + Agent + Multi-modal cùng lúc → vỡ kế hoạch.

### Bài tập 6.1

Liệt kê 3 ý tưởng prototype AI có thể build trong 1 ngày. Tự đánh giá: feasibility 1-5, novelty 1-5, demo-ability 1-5.

---

## 2. Tech stack tối thiểu — Cấp độ trung cấp

### Stack khuyến nghị (full-stack JS/TS)

```
┌────────────────────────────────────────┐
│ Frontend:   Next.js + Tailwind          │
│ Backend:    Next.js API routes / FastAPI│
│ LLM:        Anthropic / OpenAI / Groq   │
│ Vector DB:  ChromaDB (local) hoặc Pinecone│
│ Deploy:     Vercel / Railway / Fly.io   │
└────────────────────────────────────────┘
```

### Stack Python AI-focused

```
- Python 3.11+
- FastAPI / Streamlit / Gradio
- anthropic / openai SDK
- ChromaDB / Qdrant (vector)
- LangChain (nếu cần chain phức tạp)
- Docker (nếu deploy)
```

> 💡 **Mẹo — Gradio/Streamlit cho hackathon**
>
> **Gradio** = 50 dòng Python → UI chat đẹp. Cực phù hợp hackathon.
> **Streamlit** = 100 dòng → dashboard + chat + upload file. Demo data app cực nhanh.

### Hackathon starter (Streamlit + Claude)

```python
# app.py
import streamlit as st
from anthropic import Anthropic

st.set_page_config(page_title="My AI Prototype", page_icon="🤖")
st.title("Trợ lý tóm tắt báo cáo")

client = Anthropic()

if "messages" not in st.session_state:
    st.session_state.messages = []

for m in st.session_state.messages:
    st.chat_message(m["role"]).write(m["content"])

if prompt := st.chat_input("Dán báo cáo vào đây..."):
    st.session_state.messages.append({"role": "user", "content": prompt})
    st.chat_message("user").write(prompt)

    with st.chat_message("assistant"):
        with st.spinner("Đang tóm tắt..."):
            resp = client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=1024,
                system="Tóm tắt tài liệu thành 5 bullet ngắn gọn, có insight.",
                messages=[{"role": "user", "content": prompt}],
            )
            answer = resp.content[0].text
            st.write(answer)
            st.session_state.messages.append({"role": "assistant", "content": answer})
```

Chạy: `streamlit run app.py` → http://localhost:8501. **5 phút** đã có app chạy.

### Bài tập 6.2

1. Cài Streamlit, chạy code trên với API key Claude/OpenAI.
2. Thêm widget upload file PDF, dùng `pypdf` extract text, đưa vào prompt.
3. Thêm slider "độ dài tóm tắt" (3 bullet, 5 bullet, 10 bullet).

---

## 3. Build prototype thực tế — Cấp độ nâng cao

Ta sẽ build: **"Trợ lý phỏng vấn"** — upload CV, nhận câu hỏi phỏng vấn phù hợp.

### 3.1 Spec ngắn gọn

```
Input:  PDF CV + Job Description (text)
Output: 10 câu hỏi phỏng vấn cá nhân hoá + tip trả lời
Bonus:  Mock interview chat
```

### 3.2 Architecture

```
[User] → [Streamlit UI]
              │
              ├─► [PDF Parser] ──► text
              │
              └─► [LLM Prompt]
                     │
                     └─► [Claude API]
                            │
                            └─► JSON questions
```

### 3.3 Code

```python
# interview_helper.py
import streamlit as st
import json
import pypdf
from anthropic import Anthropic
from io import BytesIO

client = Anthropic()
st.title("🎤 Trợ lý phỏng vấn")

cv_file = st.file_uploader("Upload CV (PDF)", type="pdf")
jd_text = st.text_area("Job Description", height=150)

def extract_pdf(file_bytes):
    reader = pypdf.PdfReader(BytesIO(file_bytes))
    return "\n".join(p.extract_text() or "" for p in reader.pages)

PROMPT = """Bạn là HR senior. Cho CV và JD, sinh 10 câu hỏi phỏng vấn phù hợp.
Trả về JSON:
{
  "questions": [
    {"q": "...", "why": "...", "tip": "..."},
    ...
  ]
}
"""

if st.button("Tạo câu hỏi") and cv_file and jd_text:
    cv_text = extract_pdf(cv_file.read())
    with st.spinner("Đang sinh câu hỏi..."):
        resp = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            system=PROMPT,
            messages=[{"role": "user", "content": f"<cv>{cv_text}</cv>\n<jd>{jd_text}</jd>"}],
        )
    data = json.loads(resp.content[0].text)
    for i, q in enumerate(data["questions"], 1):
        with st.expander(f"Q{i}: {q['q']}"):
            st.write(f"**Vì sao hỏi:** {q['why']}")
            st.info(f"💡 **Tip:** {q['tip']}")
```

Đã có app **chạy được** trong < 60 dòng Python. Đây là phong cách hackathon.

### Bài tập 6.3

1. Build prototype trên, demo cho 1 người bạn.
2. Thêm **Mock interview**: nhấn "Thực hành" → chat với AI đóng vai interviewer.
3. Deploy lên [Streamlit Cloud](https://streamlit.io/cloud) (free).

---

## 4. Demo Day — kỹ năng pitch

### 4.1 Cấu trúc 3-phút demo

```
0:00 - 0:30  HOOK
  "Mỗi năm ở VN có 1 triệu sinh viên ra trường, mà 60% rớt phỏng vấn vì không
   biết chuẩn bị. Tôi build [tên app]."

0:30 - 1:30  DEMO LIVE
  Upload CV → JD → bấm nút → câu hỏi xuất ra → mock interview ngắn.

1:30 - 2:00  TECH (siêu ngắn)
  "Stack: Next.js + Claude + Streamlit. Latency 2s. Cost $0.02/run."

2:00 - 2:30  WHY MATTERS
  "Phù hợp với career platform, edtech, HR SaaS."

2:30 - 3:00  CALL TO ACTION
  "Demo tại link... Tôi đang tìm 2 design partner."
```

### 4.2 Mẹo demo

> 💡 **Mẹo**
>
> - **Quay video backup** trước. Mạng hội trường thường tệ.
> - **Test với data thật** đã chuẩn bị — không nhập tay khi đang demo.
> - **Click chuột mạnh**, nói chậm.
> - **Đừng xin lỗi** nếu bug nhỏ — đi tiếp.
> - **Mang USB-C / HDMI** adapter.

### 4.3 Anti-pattern khi pitch

- "Đây là Uber + AI + Web3 cho ngành làm bánh" (quá nhiều buzzword)
- Slide 30 phút trước khi demo
- Code live → mạng/IDE lỗi
- "Tương lai sẽ có..." — judge chỉ tin cái đang chạy
- Quên show **user value**

---

## 5. Mẫu pitch template

```markdown
## [Tên product]

**Problem**: Ai đau? Đau như thế nào? Có bao nhiêu người đau?

**Solution**: 1 câu mô tả cách giải.

**Demo**: (mở trình duyệt, show 30 giây use case chính)

**Why now**: Vì sao AI bây giờ mới làm được? (model rẻ hơn 10x,
context dài 1M, agent SDK chín muồi...)

**Market**: TAM/SAM. Ai sẽ trả tiền?

**Team**: Ai làm? Vì sao là người phù hợp?

**Ask**: Xin gì? (data partner, fund, mentor)
```

---

## 6. Checklist trước demo

- [ ] App chạy được offline (mock data) nếu wifi chết
- [ ] Video demo 60s đã quay sẵn
- [ ] Slide tối đa 5 cái
- [ ] Đã test trên laptop khác
- [ ] Adapter HDMI/USB-C trong túi
- [ ] Tài khoản demo có sẵn data
- [ ] API key có credit dư
- [ ] Tập pitch ≥ 3 lần với timer

---

## 7. Tóm tắt

- **Scope hẹp, demo được** > Scope rộng, không xong.
- Stack hackathon: **Streamlit/Gradio + LLM API** — 50-100 dòng đủ một MVP.
- Demo: **HOOK → DEMO → TECH → WHY → CTA** trong 3 phút.
- **Backup video** là cứu cánh khi mạng/API fail.
- Tránh buzzword salad, tránh slide quá dài.

## 8. Bài tập tổng hợp

1. **1-day sprint**: Chọn 1 ý từ bài tập 6.1, dành thực sự 8 giờ build. Cuối ngày deploy lên Streamlit Cloud / Vercel.
2. **Pitch**: Quay video 3 phút theo template phần 5. Xem lại, ghi 3 điểm cần cải thiện.
3. **Peer review**: Demo cho 2 người không cùng ngành. Họ hiểu trong 30s không?
4. **Iteration**: Sau feedback, dành 4 giờ improve. Demo lại.
5. **Đọc**:
   - [The Hackathon Survival Guide (YC)](https://www.ycombinator.com/library)
   - [Demo Day Pitch Template](https://www.ycombinator.com/library/6P-how-to-design-a-better-pitch-deck)

---

> Hết Day 6. Mai bắt đầu **Data Foundations** cho RAG: embedding, chunking, vector store.
