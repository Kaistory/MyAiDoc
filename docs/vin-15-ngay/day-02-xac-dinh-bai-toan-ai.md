---
title: Day 2 — Xác định Bài toán cho AI
sidebar_position: 2
---

# Day 2 — Xác định Bài toán cho AI

> **Mục tiêu**: Hiểu khi nào nên/không nên dùng AI, biết phân loại bài toán AI, áp dụng framework đánh giá khả thi để khỏi tốn tiền và thời gian.

## 1. Khi nào nên dùng AI? — Cấp độ cơ bản

Không phải bài toán nào cũng cần AI. Quy tắc đơn giản:

| Nên dùng AI khi... | Không nên khi... |
|---------------------|------------------|
| Dữ liệu **không có cấu trúc** (text, ảnh, audio) | Logic đơn giản, rule-based đủ |
| Đầu ra **không yêu cầu chính xác 100%** | Cần tính toán **chính xác tuyệt đối** (vd: tài chính, thuế) |
| Bài toán có nhiều **ngoại lệ**, viết rule không xuể | Bài toán **deterministic** rõ ràng |
| Cần **tự nhiên hoá** giao diện (chat, voice) | Đã có API/quy trình ổn |
| Có **đủ dữ liệu** hoặc model pre-train đủ tốt | Không có dữ liệu, không có model phù hợp |

:::tip Quy tắc 80/20
Nếu bạn có thể viết **rule** giải quyết 80% bài toán trong 1 ngày, đừng dùng AI cho phần đó.
Dùng AI để xử lý **20% phần khó** thôi — chi phí thấp, dễ debug.
:::

### Ví dụ cụ thể

✅ **Phù hợp**:
- Phân loại email spam (hàng triệu mẫu, ngoại lệ vô tận)
- Chatbot hỗ trợ khách hàng (ngôn ngữ tự nhiên)
- Tóm tắt báo cáo dài
- Sinh code, gợi ý code

❌ **Không phù hợp** (hoặc cần cẩn thận):
- Tính lương cho nhân viên (cần đúng từng đồng)
- Kiểm tra số CMND/CCCD hợp lệ (regex là đủ)
- Sắp xếp danh sách theo tên (sort là đủ)

### Bài tập 2.1

Liệt kê **5 bài toán** trong công việc hiện tại của bạn. Đánh dấu cái nào phù hợp với AI, cái nào không, kèm lý do.

---

## 2. Phân loại bài toán AI — Cấp độ trung cấp

### 2.1 Theo loại đầu vào/đầu ra

```
┌─────────────────────────────────────────────────────┐
│            Loại bài toán AI                          │
├──────────────────┬──────────────────────────────────┤
│ Classification   │ Phân email: spam/ham            │
│ Regression       │ Dự đoán giá nhà                 │
│ Generation       │ Sinh text, ảnh, code            │
│ Extraction       │ Trích thông tin từ PDF          │
│ Summarization    │ Tóm tắt báo cáo                 │
│ Translation      │ Dịch ngôn ngữ                   │
│ Search/Retrieval │ Tìm tài liệu liên quan (RAG)    │
│ Recommendation   │ Gợi ý sản phẩm                  │
│ Agent / Planning │ Tự lập kế hoạch & gọi tool      │
└──────────────────┴──────────────────────────────────┘
```

### 2.2 Theo cách tiếp cận

- **Rule-based**: viết if-else.
- **ML truyền thống**: Logistic Regression, Random Forest, XGBoost. Cần feature engineering.
- **Deep Learning custom**: train mạng nơ-ron riêng. Cần data lớn.
- **LLM zero-shot/few-shot**: prompt thẳng vào LLM, không cần train.
- **LLM + RAG**: LLM + nguồn tài liệu riêng.
- **LLM fine-tune**: huấn luyện thêm trên data của mình.
- **LLM + Agent**: LLM có khả năng gọi tool, tự lập kế hoạch.

```
        Rule-based ──────► ML ─────► DL custom ─────► LLM
        ↑ Đơn giản                                 ↑ Mạnh
        ↑ Rẻ                                       ↑ Đắt
        ↑ Dễ debug                                 ↑ Khó debug
```

:::tip Bắt đầu từ đơn giản
**Đừng nhảy thẳng vào LLM**. Thử rule-based hoặc ML cổ điển trước. Nếu không đạt thì leo thang.
:::

### Bài tập 2.2

Với bài toán "Phân loại review sản phẩm: tốt/xấu/trung tính", thiết kế:
1. Cách rule-based.
2. Cách ML cổ điển (gợi ý feature).
3. Cách LLM zero-shot (viết prompt).
4. So sánh ưu/nhược 3 cách trên về **độ chính xác, chi phí, thời gian phát triển**.

---

## 3. Framework đánh giá khả thi — Cấp độ nâng cao

Trước khi bắt tay làm, hãy chạy qua **5 câu hỏi** sau:

### 3.1 Khung 5 chữ V

| Chữ V | Câu hỏi | Ví dụ red flag |
|-------|---------|----------------|
| **Value** | Bài toán này tạo giá trị bao nhiêu? | "Để tự động hoá tiết kiệm 1 phút/ngày của 1 người" |
| **Volume** | Có bao nhiêu dữ liệu / request/ngày? | < 10 request/ngày — chưa cần AI |
| **Velocity** | Cần trả lời trong bao lâu? | Real-time < 100ms → khó với LLM lớn |
| **Variety** | Dữ liệu có nhiều biến thể không? | Chỉ 3 case → if-else đủ |
| **Veracity** | Yêu cầu độ chính xác bao nhiêu? | 99.99% → LLM một mình không đủ |

### 3.2 ROI Worksheet

```
Lợi ích/năm (USD) = (số giờ tiết kiệm × giá giờ × số người)
                  + (tăng doanh thu, conversion, retention)
                  + (giảm chi phí lỗi)

Chi phí/năm (USD) = Dev cost (lương team × tháng làm)
                  + Infra cost (API LLM, vector DB, cloud)
                  + Maintenance (10-30% dev cost/năm)
                  + Risk cost (lỗi, bồi thường, compliance)

ROI = (Lợi ích - Chi phí) / Chi phí
```

**Nguyên tắc thực dụng**: ROI > 3x ở năm đầu mới đáng làm.

### 3.3 Decision Tree

```
                Bài toán mới có chữ "tự động"
                          │
              ┌───────────┴───────────┐
            Yes                       No
              │                       │
   Dữ liệu có cấu trúc?       Không cần AI
       │
   ┌───┴───┐
  Có      Không
   │       │
  ML/SQL  Text/ảnh/audio?
            │
        ┌───┴───┐
       Có      Không
        │       │
     LLM/CV   Không cần AI
        │
   Cần kiến thức domain?
        │
   ┌────┴────┐
  Có        Không
   │         │
  RAG     LLM thẳng
   │
  Cần tool?
   │
 ┌─┴─┐
Có  Không
 │   │
Agent RAG
```

### Bài tập 2.3

Chọn 1 bài toán từ bài tập 2.1, áp dụng:
1. Khung 5V để đánh giá.
2. Ước tính ROI thô (dev cost = lương 1 dev × 2 tháng, infra ~ $200/tháng).
3. Đi qua decision tree — kết luận: rule-based / ML / LLM / RAG / Agent?

---

## 4. Anti-pattern thường gặp

:::warning Sai lầm phổ biến
1. **AI-first**: "Phải có AI" rồi mới tìm bài toán → ngược.
2. **Over-engineer**: Bài toán đơn giản nhưng dùng multi-agent + vector DB + fine-tune.
3. **Hallucination tolerance**: Dùng LLM cho việc cần đúng 100% (tài chính, y tế) mà không có guardrail.
4. **No baseline**: Không có rule/ML baseline để so sánh — không biết AI có thực sự cải thiện không.
5. **No eval set**: Không có bộ test, đánh giá bằng cảm tính.
:::

## 5. Phương pháp Product Discovery cho AI

**Quy trình 6 bước** trước khi viết dòng code đầu tiên:

1. **Discover**: phỏng vấn user — họ đau ở đâu, đang làm thủ công ra sao?
2. **Quantify**: ước tính giá trị (giờ tiết kiệm, doanh thu).
3. **Baseline**: thử rule-based / ML đơn giản — đo accuracy.
4. **Spike**: prototype với LLM trong 1-2 ngày.
5. **Compare**: so sánh baseline vs AI — chênh lệch có đáng không?
6. **Decide**: go/no-go, hay đợi đến khi data đủ.

---

## 6. Tóm tắt

- Dùng AI khi: data **không cấu trúc**, **nhiều ngoại lệ**, cần **tự nhiên hoá**, chấp nhận sai số.
- **Đừng bỏ qua rule-based / ML cổ điển** — chúng rẻ, nhanh, dễ debug.
- 5 chữ V: **Value, Volume, Velocity, Variety, Veracity**.
- Luôn có **baseline** để biết AI cải thiện được bao nhiêu.
- Tính **ROI** thô trước khi bắt đầu.

## 7. Bài tập tổng hợp

1. **Phỏng vấn**: Hỏi 2 đồng nghiệp về "việc lặp đi lặp lại mỗi ngày". Chọn 1 việc, làm worksheet đánh giá khả thi AI.
2. **Reverse engineering**: Chọn 3 sản phẩm AI bạn dùng (vd: Gmail Smart Reply, Notion AI, GitHub Copilot). Đoán: họ dùng LLM, rule, hay hybrid?
3. **Case study**: Đọc bài [How GitHub Copilot was built](https://github.blog/2023-05-17-how-github-copilot-is-getting-better-at-understanding-your-code/). Tìm xem họ quyết định **không** dùng AI ở chỗ nào.
4. **Viết PRD ngắn** (1 trang) cho bài toán bạn chọn, gồm: Problem, User, Value, Solution, Risks.

---

> Hết Day 2. Mai sẽ học **ReAct pattern** — kiến trúc Agent cơ bản nhất.
