---
title: Day 2 — Xác định Bài toán cho AI
sidebar_position: 2
---

# Day 2 — Xác định Bài toán cho AI

> **Mục tiêu**: Hiểu khi nào nên/không nên dùng AI, biết phân loại bài toán AI, áp dụng framework đánh giá khả thi để khỏi tốn tiền và thời gian.

## 0. Slide bài giảng

<div class="slide-deck" id="day2-deck">
  <div class="slide-stage">
    <img class="slide-img" id="day2-img" src="Vin%20AI/Day-2/Slide/Screenshot_256.png" alt="Slide 1" />
  </div>
  <div class="slide-controls">
    <button type="button" class="slide-btn" id="day2-prev" aria-label="Slide trước">← Trước</button>
    <span class="slide-counter" id="day2-counter">1 / 32</span>
    <button type="button" class="slide-btn" id="day2-next" aria-label="Slide kế">Sau →</button>
  </div>
  <div class="slide-hint">Dùng phím ← → để chuyển slide. Bấm vào ảnh để phóng to.</div>
</div>

<style>
.slide-deck { border: 1px solid #e3e3e3; border-radius: 10px; padding: 14px; margin: 16px 0 24px; background: #fafafa; }
.slide-stage { display: flex; justify-content: center; align-items: center; background: #111; border-radius: 6px; overflow: hidden; min-height: 320px; }
.slide-img { max-width: 100%; max-height: 72vh; object-fit: contain; display: block; cursor: zoom-in; }
.slide-controls { display: flex; justify-content: space-between; align-items: center; gap: 12px; margin-top: 12px; }
.slide-btn { background: #2c7be5; color: #fff; border: none; padding: 8px 16px; border-radius: 6px; cursor: pointer; font-size: 14px; font-weight: 600; }
.slide-btn:hover { background: #1a5db8; }
.slide-counter { font-weight: 700; color: #333; font-variant-numeric: tabular-nums; }
.slide-hint { text-align: center; font-size: 12px; color: #888; margin-top: 8px; }
</style>

<script>
(function () {
  var total = 32;
  var startNum = 256;
  var basePath = 'Vin%20AI/Day-2/Slide/Screenshot_';
  var current = 0;
  var img = document.getElementById('day2-img');
  var counter = document.getElementById('day2-counter');
  var prevBtn = document.getElementById('day2-prev');
  var nextBtn = document.getElementById('day2-next');
  if (!img || !counter || !prevBtn || !nextBtn) return;

  function render() {
    img.src = basePath + (startNum + current) + '.png';
    img.alt = 'Slide ' + (current + 1);
    counter.textContent = (current + 1) + ' / ' + total;
  }
  function prev() { current = (current - 1 + total) % total; render(); }
  function next() { current = (current + 1) % total; render(); }

  prevBtn.addEventListener('click', prev);
  nextBtn.addEventListener('click', next);

  function keyHandler(e) {
    if (!document.getElementById('day2-deck')) {
      document.removeEventListener('keydown', keyHandler);
      return;
    }
    if (e.target && /^(INPUT|TEXTAREA)$/.test(e.target.tagName)) return;
    if (e.key === 'ArrowLeft') prev();
    else if (e.key === 'ArrowRight') next();
  }
  document.addEventListener('keydown', keyHandler);

  render();
})();
</script>

---

## 1. Khi nào nên dùng AI? — Cấp độ cơ bản

Không phải bài toán nào cũng cần AI. Quy tắc đơn giản:

| Nên dùng AI khi... | Không nên khi... |
|---------------------|------------------|
| Dữ liệu **không có cấu trúc** (text, ảnh, audio) | Logic đơn giản, rule-based đủ |
| Đầu ra **không yêu cầu chính xác 100%** | Cần tính toán **chính xác tuyệt đối** (vd: tài chính, thuế) |
| Bài toán có nhiều **ngoại lệ**, viết rule không xuể | Bài toán **deterministic** rõ ràng |
| Cần **tự nhiên hoá** giao diện (chat, voice) | Đã có API/quy trình ổn |
| Có **đủ dữ liệu** hoặc model pre-train đủ tốt | Không có dữ liệu, không có model phù hợp |

> 💡 **Mẹo — Quy tắc 80/20**
>
> Nếu bạn có thể viết **rule** giải quyết 80% bài toán trong 1 ngày, đừng dùng AI cho phần đó.
> Dùng AI để xử lý **20% phần khó** thôi — chi phí thấp, dễ debug.

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

> 💡 **Mẹo — Bắt đầu từ đơn giản**
>
> **Đừng nhảy thẳng vào LLM**. Thử rule-based hoặc ML cổ điển trước. Nếu không đạt thì leo thang.

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

> ⚠️ **Cảnh báo — Sai lầm phổ biến**
>
> 1. **AI-first**: "Phải có AI" rồi mới tìm bài toán → ngược.
> 2. **Over-engineer**: Bài toán đơn giản nhưng dùng multi-agent + vector DB + fine-tune.
> 3. **Hallucination tolerance**: Dùng LLM cho việc cần đúng 100% (tài chính, y tế) mà không có guardrail.
> 4. **No baseline**: Không có rule/ML baseline để so sánh — không biết AI có thực sự cải thiện không.
> 5. **No eval set**: Không có bộ test, đánh giá bằng cảm tính.

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

## 8. Bốn câu hỏi trọng tâm & Ba trụ cột nền tảng

> Bối cảnh từ slide: từ yêu cầu mơ hồ ("tôi muốn xây chatbot AI") đến Problem Statement rõ ràng.

### 8.1 Bốn câu hỏi trọng tâm — Từ xác định bài toán đến quyết định ứng dụng AI

1. **Bài toán có thực sự cần AI giải quyết?**
2. Nếu có, **giải pháp ở cấp độ nào**: Rule, Workflow, hay Agent?
3. **Problem Statement đã đủ rõ ràng** để triển khai chưa?
4. **Khi nào quyết định**: Go, Not Yet, hay No-Go?

### 8.2 Ba trụ cột nền tảng của AI Product

| Trụ cột | Nguồn | Tư duy |
|---------|-------|--------|
| **AI Engineering** | Chip Huyen — *AI Engineering* | Kỹ thuật hệ thống AI: RAG, Agent, Guardrails, Evaluation; vận hành hệ thống AI thực tế |
| **Product Thinking** | Marty Cagan — *Inspired* | Tư duy sản phẩm: xác định đúng bài toán, thấu hiểu người dùng, tránh xây tính năng không giá trị |
| **Design Thinking** | Don Norman — *The Design of Everyday Things* | Tư duy thiết kế dựa trên mô hình tư duy (Mental Model), cơ chế phản hồi (Feedback), tối ưu trải nghiệm khi AI sai sót |

### 8.3 "AI chatbot" chưa phải là một bài toán

Cùng tên gọi nhưng **đối tượng khác → metric khác → rủi ro khác**:

| Phục vụ khách hàng | Hỗ trợ nội bộ |
|--------------------|----------------|
| Giải đáp FAQ về sản phẩm & chính sách | Phân loại yêu cầu hỗ trợ (Tickets/Questions) |
| Tư vấn và hỗ trợ mua hàng | Tra cứu thông tin nghiệp vụ nhanh |
| Chăm sóc sau mua hàng | Đề xuất nháp phản hồi cho con người phê duyệt |
| Bán thêm & bán chéo (Upsell, Cross-sell) | Chuyển tiếp câu hỏi phức tạp/rủi ro cao cho nhân sự |

> 💡 **Mẹo — Khoan đã, bạn có hỏi không?**
>
> Trước khi đề xuất giải pháp, hỏi: **Học viên gặp khó khăn ở công đoạn nào? Trợ giảng quá tải ở bước nào? Quy trình hiện tại đang xử lý ra sao? Giải pháp này xây dựng phục vụ ai?** Chưa thấu hiểu **pain point** thì chưa đề xuất giải pháp.

---

## 9. Problem Discovery — Tìm đúng vấn đề trước khi tìm giải pháp

> *"Never solve the problem I am asked to solve."* — **Don Norman**, *The Design of Everyday Things*
>
> Giải pháp xuất sắc cho sai vấn đề có thể còn tệ hơn không có giải pháp.

### 9.1 Mô hình Double Diamond (Don Norman / British Design Council, 2005)

```
   FINDING THE RIGHT          FINDING THE RIGHT
       PROBLEM                    SOLUTION
   ◇─────────◇                ◇─────────◇
  Discover  Define          Develop   Deliver
   (mở rộng) (thu hẹp)       (mở rộng) (thu hẹp)
```

- **Diamond 1 — Tìm đúng vấn đề**
  - *Discover*: mở rộng — khảo sát vấn đề căn bản
  - *Define*: thu hẹp — xác định đúng bài toán gốc
- **Diamond 2 — Tìm đúng giải pháp**
  - *Develop*: mở rộng — nhiều giải pháp tiềm năng
  - *Deliver*: thu hẹp — chọn và triển khai

### 9.2 Diamond 1 — Khám phá & Định nghĩa

| Khám phá *(mở rộng)* | Định nghĩa *(thu hẹp)* |
|----------------------|------------------------|
| Quan sát thực tế (Observation) | Lập bản đồ tương đồng (Affinity Mapping) |
| Phỏng vấn người dùng (User interview) | Kỹ thuật đặt câu hỏi 5 Whys |
| Khảo sát (Survey) | Ma trận tác động — nỗ lực (Effort Matrix) |
| Nhật ký hành vi (Diary Study) | Biểu quyết bằng chấm dán (Dot Voting) |
| Phân tích dữ liệu / Nhật ký hệ thống | Câu hỏi mở hướng giải quyết (How Might We) |
| Bản đồ các bên liên quan (Stakeholder Mapping) | Phát biểu bài toán (Problem Statement) |

### 9.3 HCD — Quy trình thiết kế lấy con người làm trung tâm (4 bước)

1. **Quan sát** — Quan sát thực tế, không phỏng vấn xem chuyện gì đang xảy ra
2. **Ideation** — Tạo ý tưởng, sàng lọc giải pháp tiềm năng dựa trên quan sát
3. **Prototype** — Tạo mô hình thử nhanh để kiểm thử ý tưởng
4. **Test** — Người dùng thực sử dụng, đo lường, lặp lại

### 9.4 Những câu hỏi nguyên bản — Ví dụ kinh điển

| Ai | Câu hỏi nguyên bản |
|----|--------------------|
| **Isaac Newton** | "Tại sao quả táo rơi xuống đất?" (không phải "Vì sao Mặt Trăng ở trên trời?") |
| **Polaroid** | "Tại sao chúng ta phải chờ để xem ảnh?" |
| **Airbnb** | "Liệu mọi người có sẵn sàng ngủ ở nhà người lạ?" |

> 💡 **Mẹo — Câu hỏi gợi mở**
>
> - Giả định hiện nhiên nào cần được lật lại?
> - Có cách tiếp cận nào hoàn toàn mới cho vấn đề?
> - Nếu thiết kế lại từ đầu và không bị giới hạn?
> - Tại sao bài toán này cần AI? Nếu không thì sao?
> - Quy trình nào đang tồn tại chỉ vì thói quen?
> - Có câu hỏi cốt lõi nào đang bị né tránh?

### 9.5 Tìm bài toán AI ở đâu — 4 Lenses

Bắt đầu từ việc quan sát các hoạt động thực tế xung quanh:

| Lăng kính | Dấu hiệu |
|-----------|----------|
| **Repetitive** — Tác vụ lặp lại | Việc diễn ra thường xuyên, công đoạn nào cần chuẩn hóa và hướng tới tự động hóa |
| **Time-consuming** — Tiêu tốn thời gian | Khối lượng xử lý lớn, thời gian dài; mọi người phải bỏ vào (tìm kiếm, đọc hiểu, chờ đợi, định dạng) |
| **AI advantage** — Lợi thế của AI | Tác vụ đòi hỏi phân tích ngữ cảnh, xử lý ngôn ngữ tự nhiên, tổng hợp đa nguồn |
| **User pain points** — Điểm đau người dùng | Ai đang gặp khó khăn, phàn nàn hoặc bị tắc nghẽn liên tục? |

> Tập trung **nhận diện vấn đề**; chưa vội đề xuất giải pháp. Sàng lọc bài toán sẽ diễn ra vào buổi chiều.

### 9.6 Khởi nguồn từ bài toán, không bắt đầu từ AI — Case Study

| Sản phẩm | Bài học |
|----------|---------|
| **Cursor** | Lệch năng lực cốt lõi — từ bỏ mảng AI thiết kế cơ khí để tập trung vào AI code editor; nội đội ngũ am hiểu sâu sắc quy trình nghiệp vụ |
| **Artifact** | Sản phẩm tốt ≠ Thị trường lớn — ứng dụng đọc tin tích hợp AI xuất sắc, nhưng quy mô thị trường quá hẹp để thương mại hóa thành công |
| **NotebookLM** | Định vị đúng điểm đau — tập trung giải quyết nhu cầu cần đáp, tóm tắt trên tài liệu cá nhân và đối chiếu nguồn gốc bằng trích dẫn |

**Lộ trình**: Bài toán → Quy trình vận hành → Chỉ số đo lường → Giải pháp AI

### 9.7 Discovery Interview — 5 câu hỏi nên hỏi stakeholder

1. **Vấn đề nhức nhối (Pain Point) là gì?** Tần suất lặp lại trong ngày hoặc trong tuần ra sao?
2. **Quy trình (Workflow) hiện tại như thế nào?** Công cụ nào đang sử dụng ở từng bước, và ai bàn giao công việc cho ai?
3. **Thiệt hại (Cost) đo lường là gì?** Hao phí cụ thể về thời gian xử lý, chi phí tài chính, cam kết dịch vụ (SLA) hay tỷ lệ chuyển đổi (conversion)?
4. **Hậu quả nếu hệ thống AI sai sót là gì?** Khâu nào cần con người tham gia kiểm soát (HITL/phê duyệt), hay AI chỉ hỗ trợ đưa ra gợi ý?
5. **Ai là người có quyền phê duyệt dự án (nói YES)?** Chỉ số hiệu quả (metric) và mức độ rủi ro (risk) nào sẽ trực tiếp quyết định việc đầu tư?

> ⚠️ Nếu đối tác (stakeholder) không mô tả được quy trình hiện tại và chi phí thiệt hại khi xảy ra, mọi đề xuất AI đầu chỉ là phỏng đoán thiếu căn cứ.

---

## 10. Quick Problem Card — Khung định hình bài toán

Khung **6 trường** dùng để diễn đạt một bài toán trước khi quyết định cấp độ giải pháp:

| Trường | Mô tả |
|--------|-------|
| **Bài toán** *(problem)* — 1 câu | Vấn đề cụ thể cần giải quyết (không bao gồm giải pháp) |
| **Đối tượng ảnh hưởng** *(actor)* | Cá nhân hoặc bộ phận chịu tác động trực tiếp từ vấn đề |
| **Quy trình hiện tại** *(workflow)* | Các bước vận hành thủ công hoặc tự động hiện tại (gồm 3–7 bước) |
| **Nút thắt & Tác động** *(bottleneck + impact)* | Khâu gây chậm trễ, sai sót hoặc lặp lại; hệ quả hay tổn thất cụ thể |
| **Chỉ số đo thành công** *(success metric)* | Chỉ số định lượng cụ thể để chứng minh hiệu quả cải tiến |
| **Định hướng giải pháp** *(direction)* | No AI / Rule / Workflow / Agent / Chưa xác định |

### 10.1 Ví dụ đã điền — Case: Weekly Report

| Trường | Nội dung |
|--------|----------|
| **Bài toán** | Mỗi chiều Hai, PM mất khoảng 90 phút tổng hợp Weekly Report từ Jira, Google Sheets và Slack; bước viết narrative tốn thời gian nhất và dễ làm trễ deadline |
| **Đối tượng ảnh hưởng** | Junior PM chịu trách nhiệm gửi weekly report cho Engineering Manager và CEO trước buổi leadership sync |
| **Quy trình hiện tại** | Export Jira → lấy metrics từ Google Sheets → đọc Slack recap → tổng hợp vào Google Docs → viết narrative → review/format → gửi email |
| **Nút thắt & Tác động** | Bước viết narrative từ raw data mất khoảng 25 phút. Tổng flow mất khoảng 90 phút/tuần/PM; team 3 PM tương đương khoảng 270 phút/tuần |
| **Chỉ số thành công** | Giảm thời gian làm report từ 90 phút xuống dưới 30 phút, nhưng không làm tăng số câu CEO/EM phải hỏi lại |
| **Định hướng giải pháp** | **Workflow** — tự động kéo và cấu trúc dữ liệu, AI hỗ trợ draft narrative, PM vẫn review/edit trước khi gửi |

### 10.2 Câu hỏi khai thác bài toán (6 câu định hình)

1. Quy trình hiện tại như thế nào? Công cụ, các bước, cơ chế bàn giao thông tin?
2. Nút thắt nằm ở đâu? Bước nào chậm, dễ sai hoặc lặp lại?
3. Hao phí hiện tại là bao nhiêu? Thời gian, chi phí nhân sự, SLA, cơ hội bỏ lỡ?
4. Tiêu chí thành công đo bằng gì? Hiệu quả cải tiến định lượng cụ thể?
5. Hậu quả khi xảy ra sai sót? Phạm vi tự quyết của AI; điểm cần con người phê duyệt?
6. Có giải pháp phi AI đơn giản hơn? Quy tắc, checklist, quy trình hay tài liệu hướng dẫn?

---

## 11. Định lượng hóa bài toán

> **Điểm đau chưa được định lượng thì không thể xác định giá trị thực tế của AI.**

### 11.1 Khung Baseline → Target → Measurement

| Bước | Câu hỏi | Ví dụ (Weekly Report) |
|------|---------|------------------------|
| **01 — Baseline** *(where we are)* | Mức nào hiện tại là bao nhiêu? Bằng con số cụ thể? | 90 phút/tuần/PM; tỉ lệ lỗi phân loại 20% |
| **02 — Target** *(where to go)* | Kỳ vọng cải thiện ở mức độ nào? Ngưỡng cụ thể là gì? | Dưới 30 phút; tỉ lệ lỗi dưới 5% |
| **03 — Measurement** *(how we know)* | Chỉ số nào chứng minh tính hiệu quả? Cách thu thập? | Đo thời gian thực tế khi PM gửi report; đối chiếu Slack feedback |

### 11.2 Thiết lập chỉ số — Output & Input metrics

Chỉ số đo lường cần phản ánh **kết quả cuối** và **các đòn bẩy** có thể tác động:

| Output Metric *(what we optimize)* | Input Metric *(what we can move)* |
|-------------------------------------|-----------------------------------|
| Kết quả cuối cùng | Các đòn bẩy |
| Thời lượng hoàn tất quy trình giảm bao nhiêu? | Tỷ lệ câu hỏi được phân loại chính xác |
| Tỷ lệ sai sót / Chất lượng đầu ra thay đổi thế nào? | Tỷ lệ yêu cầu được chuyển tiếp hỗ trợ kịp thời |
| Giá trị thực tế người dùng nhận được rõ nét hơn? | Thời gian Trợ giảng kiểm bản nháp phản hồi |

> "Nâng cao hiệu suất" **không phải chỉ số** — cần gắn với hiện trạng, mục tiêu và phương pháp đo.

---

## 12. Có nên ứng dụng AI?

AI chỉ thực sự mang lại giá trị khi tích hợp chính xác vào quy trình nghiệp vụ và giải quyết đúng điểm đau.

### 12.1 Khi nào AI đáng để làm?

| AI hợp khi nào | Vì sao doanh nghiệp đầu tư |
|-----------------|----------------------------|
| Tác vụ mang tính lặp lại nhưng có độ biến thiên vừa phải | **01 — Sống còn**: bắt buộc phải tích hợp AI để duy trì lợi thế cạnh tranh trước đối thủ |
| Yêu cầu tổng hợp hoặc tìm kiếm từ nhiều nguồn | **02 — Hiệu quả**: giảm thiểu chi phí vận hành, tăng tốc độ xử lý và nâng cao năng suất nghiệp vụ |
| Quy trình gồm nhiều bước phức tạp và cần tương tác với nhiều công cụ | **03 — Khám phá**: tích lũy năng lực công nghệ, tránh tụt hậu và tìm kiếm các mô hình cơ hội mới |
| *Nếu quy trình hoàn toàn có tính xác định (deterministic), các quy tắc luật tĩnh (rule) sẽ tối ưu hơn* | |

### 12.2 Tự xây dựng hay mua giải pháp? (Build / Boost / Buy)

**Góc nhìn 1 — Chip Huyen, AI Engineering (2025):**

| In-house (Build) | Mua / SaaS (Buy) |
|------------------|------------------|
| Khi công nghệ AI là lợi thế cạnh tranh cốt lõi và yếu tố sống còn | Khi giải pháp AI đóng vai trò như một công cụ tối ưu hóa năng suất (productivity layer) |

**Góc nhìn 2 — MIT CISR (2025):**

| Buy | Boost | Build |
|-----|-------|-------|
| Giải pháp may sẵn (off-the-shelf), do nhà cung cấp (vendor) duy trì | Mua mô hình sẵn có và cải tiến bằng dữ liệu nội bộ | Tự xây dựng và tối ưu mô hình tùy biến (custom model) riêng |
| Triển khai nhanh, nhưng ít tạo ra sự khác biệt cạnh tranh | Ứng dụng kỹ thuật tinh chỉnh (fine-tune) hoặc RAG (truy xuất tăng cường) | Khả năng kiểm soát cao nhất, nhưng chi phí đắt đỏ nhất |
| Phụ thuộc hoàn toàn vào lộ trình (roadmap) của vendor | Đòi hỏi năng lực quản trị dữ liệu (data governance) tốt | Đòi hỏi đội ngũ kỹ sư AI có năng lực chuyên môn mạnh |

> **Thực tế**: Đa số đội ngũ phát triển đang ở giữa — **Boost** (RAG / fine-tune), thay vì phải tự xây dựng lại mọi thứ từ đầu (build from scratch).

### 12.3 Thiết lập kỳ vọng — 3 nhóm chỉ số

| 1 — Tác động kinh doanh | 2 — Sự hài lòng khách hàng | 3 — Ngưỡng hữu dụng |
|--------------------------|----------------------------|---------------------|
| Tỷ lệ tự động hóa tác vụ (cấp%) | Chỉ số hài lòng CSAT / NPS | **Chất lượng**: độ chính xác và tính hữu ích của đầu ra |
| Quy mô xử lý công việc tăng thêm | Cảm giá chất lượng trực tiếp từ người dùng | **Độ trễ**: tốc độ phản hồi (TTFT, TPOT) |
| Tốc độ phản hồi & thời gian quy trình được tối ưu | Tỷ lệ hoàn thành tác vụ vs Tỷ lệ bỏ ngang giữa chừng | **Chi phí**: chi phí tài chính trên mỗi lượt yêu cầu |

### 12.4 Đánh giá mức độ phù hợp của AI — 5 câu hỏi cốt lõi

1. Nghiệp vụ có đòi hỏi xử lý ngôn ngữ, tri thức chuyên môn hoặc suy luận?
2. Dữ liệu đầu vào có cung cấp đủ ngữ cảnh để AI phản hồi chính xác?
3. Đã thiết lập các chỉ số định lượng để đánh giá hiệu quả chưa?
4. Hậu quả khi AI sai sót có nằm trong phạm vi kiểm soát?
5. Có giải pháp thay thế đơn giản và tối ưu chi phí hơn AI không?

> Nếu phần lớn câu trả lời **chưa rõ ràng** → Quyết định: **Not Yet**.

### 12.5 Vòng đời sản phẩm AI (AI Product Lifecycle — Chip Huyen)

```
Milestone 1: Planning & Use Case Evaluation (Quick Walk-Run, AI fit, Defensibility)
Milestone 2: Expectations & Milestone Planning (Usefulness Thresholds, Build vs. Buy)
Milestone 3: Model Selection (Hard filters, Task-specific Evals, Build vs. Buy)
Milestone 4: Architecture Evolution (RAG → Guardrails → Routing/Caching → Simple Prompt)
Milestone 5: Evaluation-Driven Development (Pre-Component → End-to-End → AI as Judge)
Milestone 6: Monitoring & Feedback Loop (Observability, Explicit + Implicit Feedback)
                                                   ↓
                                            Data Flywheel
                                                   ↓
                                       Dataset Engineering
```

### 12.6 Khoảng cách giữa Demo và Production

| Bước | Mục tiêu |
|------|----------|
| **Baseline** — Thiết lập đối chứng | Đối chiếu hiệu quả AI với quy tắc tĩnh, nhân sự hay quy trình hiện tại |
| **Evaluation** — Kiểm thử hệ thống | Bộ dữ liệu kiểm thử, kịch bản biên (edge cases) và tiêu chí nghiệm thu |
| **Controls** — Cơ chế kiểm soát | Logging, fallback, rollback và nhân sự chịu trách nhiệm |
| **Operations** — Vận hành liên tục | AI giám sát lỗi, cập nhật tri thức nền và tối ưu hệ thống |

> Mục tiêu Day 02 là **xác định tính khả thi** để tiếp tục nghiên cứu; chưa phải quyết định triển khai ngay.

### 12.7 Hệ thống AI = Model + Context + Planning + Tools

```
              ┌──────────────────────┐
              │  Orchestrator /      │
              │  System Logic        │  ←─ Loop: Self Policy
              └─────────┬────────────┘
   ┌────────────────────┼──────────────────────┐
   ▼                    ▼                      ▼
[ Model ]          [ Context ]            [ Tools ]
LLM · SLM       RAG · Memory          APIs · Search
                Wrong Retrieval       Side Effects · Security

                      [ Planning ]
                      Steps · Policies
```

**Bốn thành phần:**

| Thành phần | Vai trò |
|------------|---------|
| **Tư duy & Sáng tạo** *(Model)* | Xử lý ngôn ngữ, tri thức tổng hợp, sinh đầu ra phù hợp ngữ cảnh |
| **Tri thức chuyên biệt** *(Context)* | Dữ liệu đặc thù của tổ chức, nghiệp vụ hoặc tài liệu sản phẩm |
| **Điều phối quy trình** *(Planning)* | Tự lập kế hoạch — Pilot → Vận hành thực tế → Vòng lặp phản hồi |
| **Liên kết hệ thống** *(Tools)* | Kết nối các CRM, công cụ nội bộ, hệ thống nghiệp vụ |

### 12.8 Vai trò UX trong sản phẩm AI

**AI không cần hoàn hảo, nếu UX đỡ được chỗ nó yếu:**

| Tình huống | Giải pháp UX |
|------------|--------------|
| **Không chắc** (low confidence) | Xin user xác nhận trước khi thực hiện |
| **Risk cao** (sai = hậu quả nghiêm trọng) | Chỉ suggest, không auto-action |
| **Câu trả lời dài** (quá tải thông tin) | Chia option / card / summary cho user chọn |
| **Thiếu context** (input mơ hồ) | Hỏi lại đúng chỗ thay vì đoán sai |

> AI Product = AI + UX. Dùng UX để đỡ chỗ AI chưa đủ tốt.

---

## 13. Rule / Workflow / Agent — Ba cấp độ giải pháp

### 13.1 Tổng quan ba mức

| Rule / Script | LLM Feature / Workflow | Agent |
|---------------|------------------------|-------|
| Đầu vào ổn định, ít thay đổi | Đầu vào đa dạng, không viết hết rule được | Nhiều bước, dùng nhiều công cụ |
| Logic viết được thành if/else | Đầu ra cần linh hoạt (tóm tắt, dịch, phân loại) | Tình huống thay đổi liên tục |
| Cần kết quả đúng 100% | Có cách đo chất lượng | Cần tự ra quyết định giữa các bước |
| Quy định pháp lý / tuân thủ chặt | Người có thể kiểm tra trước khi gửi | Có kiểm soát rủi ro rõ ràng |
| **VD**: Tính thuế, chặn email spam theo từ khóa, auto-reply theo template | **VD**: Tóm tắt email, chatbot FAQ, phân loại ticket hỗ trợ | **VD**: Agent nghiên cứu thị trường, coding agent sửa nhiều file |

> Thứ tự ưu tiên thực dụng: bắt đầu từ bên trái, chỉ di sang bên phải khi giá trị tăng hơn độ phức tạp.

### 13.2 Cấp độ 1 — Rule-based (khi nào chọn)

| Khi nào chọn Rule | Ví dụ thực tế (lớp học) |
|--------------------|--------------------------|
| Logic phân nhánh rành mạch (If/Else) | Hỏi lịch nộp bài → Tự động gửi link thời khóa biểu |
| Yêu cầu hoặc trạng thái lại hoàn toàn | Nộp thiếu file bài tập → Tự động nhắc nhở checklist |
| Không đòi hỏi khả năng tư duy của AI | Hỏi cài đặt quen thuộc → Gửi link tài liệu hướng dẫn |
| Yêu cầu kết quả có thể dự đoán và kiểm soát tuyệt đối | Câu hỏi ngoài danh mục → Tự động chuyển cho Trợ giảng |

> Giải pháp dựa trên Luật (Rule) **không thua kém AI**. Nếu giải quyết triệt để bài toán, đó luôn là lựa chọn tối ưu nhất.

### 13.3 Cấp độ 2 — Workflow

Các bước xử lý đã định hình rõ, nhưng từng công đoạn cần AI hỗ trợ ngôn ngữ hoặc đánh giá:

```
Học viên gửi Problem Card  →  AI rà soát & yêu cầu bổ sung  →  Trợ giảng phê duyệt câu phức tạp
```

| Ưu điểm | Lưu ý thiết kế |
|---------|----------------|
| **Linh hoạt nhưng có kiểm soát** | **Tránh chatbot phản hồi tự do** |
| Xử lý ngữ cảnh tốt hơn Rule tĩnh | Mỗi công đoạn phải định nghĩa rõ đầu vào và đầu ra |
| Lộ trình của hệ thống vẫn nằm trong tầm kiểm soát | Không thiết kế thành một chatbot phản hồi tự do |

### 13.4 Cấp độ 3 — Agent (khi nào cân nhắc)

| Khi nào dùng Agent | Ví dụ thực tế (lớp học) |
|---------------------|--------------------------|
| Không thể xác định trước toàn bộ các bước thực thi | Theo dõi hoạt động thảo luận và nộp bài trên các kênh học tập |
| Môi trường nhiều biến số đòi hỏi thay đổi kế hoạch linh hoạt | Phát hiện các học viên hoặc nhóm học viên bị kẹt quá lâu |
| Cần tương tác với nhiều công cụ và truy xuất nhiều nguồn dữ liệu | Tự động tổng hợp vấn đề hay gặp phải và gợi ý cách hỗ trợ |
| Có thiết lập vòng phản hồi và chốt chặn giám sát từ con người | Trợ giảng chỉ cần duyệt và nhấn nút gửi phương án hỗ trợ |

> Tác động của Agent mạnh mẽ hơn, nhưng đi kèm **chi phí vận hành cao hơn, độ trễ lớn hơn, khó kiểm thử** và phát sinh các dạng lỗi phức tạp.

### 13.5 Một tình huống, ba cấp độ giải pháp

| Cấp độ 1 — Rule *(luật tĩnh)* | Cấp độ 2 — Workflow *(quy trình)* | Cấp độ 3 — Agent *(tác nhân)* |
|--------------------------------|------------------------------------|-------------------------------|
| **Trả lời tự động** | **Duyệt Problem Card** | **Đề xuất can thiệp chủ động** |
| Tự động trả lời FAQ, gửi link thời khóa biểu | AI kiểm tra đầy đủ của Problem Card | Tự động theo dõi tiến độ nộp bài |
| Gửi lại liệu sửa lỗi cài đặt cơ bản | Yêu cầu bổ sung nếu thiếu thông tin | Phát hiện nhóm học viên bị kẹt lại |
| Nhắc nhở checklist nộp bài | Chuyển cho Trợ giảng giải quyết | Chuẩn bị câu trả lời, đề xuất TA duyệt |
| **Khi nào?** Logic tường minh, kết quả cố định | **Khi nào?** Có quy trình rõ, AI hỗ trợ từng bước | **Khi nào?** Tình huống động, đa công cụ |

> Không bắt buộc nâng cấp tuần tự từ Rule lên Agent → **dừng ở cấp tối giản nhất** nếu đã đáp ứng mục tiêu đề ra.

### 13.6 Workflow Patterns theo Anthropic — Building Effective Agents (2024)

**Basic Patterns** *(đủ cho hầu hết tác vụ)*:

| Pattern | Mô tả | Ví dụ |
|---------|-------|-------|
| **Prompt Chaining** | Chia task thành chuỗi bước tuần tự. Có *gate* kiểm tra giữa các bước | Viết outline → check → viết bài |
| **Routing** | Phân loại input → đưa vào nhánh chuyên biệt. Tối ưu từng loại riêng | CS query → FAQ / refund / kỹ thuật |
| **Parallelization** | Chạy song song rồi tổng hợp (sectioning), hoặc chạy nhiều lần lấy vote | Guardrail + response đồng thời |

**Advanced Patterns** *(khi nghiệp vụ đòi hỏi)*:

| Pattern | Mô tả | Ví dụ |
|---------|-------|-------|
| **Orchestrator-Workers** | 1 LLM điều phối, 1 LLM workers. Subtasks không biết trước | Coding agent sửa nhiều file |
| **Evaluator-Optimizer** | 1 LLM tạo, 1 LLM đánh giá → lặp cho đến khi đạt | Dịch văn bản → review → sửa |

**Autonomous: Agent** — LLM tự lập kế hoạch + gọi tools + iterate. Autonomous loop. VD: SWE-bench, computer use.

> **Nguyên tắc Anthropic**: Luôn ưu tiên giải pháp đơn giản nhất; chỉ tăng độ phức tạp khi thực sự cần thiết. 2 mô hình cơ bản trên đã đủ đáp ứng hầu hết bài toán thực tế.

### 13.7 Thang câu hỏi lựa chọn cấp độ giải pháp

Khung câu hỏi tuần tự giúp **tránh bẫy "nhảy vọt"** lên Agent phức tạp:

| # | Tiêu chí | Hướng quyết định |
|---|----------|------------------|
| **01** *Tần suất & Tác động* | Tần suất & Tác động có đủ lớn? | Nếu thấp → Xử lý thủ công hoặc hiệu chỉnh quy trình nghiệp vụ trước |
| **02** *Logic* | Logic xử lý có rành mạch? | Nếu tường minh → Ưu tiên giải pháp Rule, kịch bản tự động, danh mục kiểm tra |
| **03** *Quy trình* | Quy trình thực hiện có cố định? | Nếu có → Xây dựng Workflow tích hợp AI hỗ trợ từng công đoạn |
| **04** *Tự thích ứng* | Quy trình đòi hỏi khả năng thích ứng linh hoạt? | Chỉ khi có nhiều biến số phức tạp → Mới cần nhấc Agent |
| **05** *Giá trị vs rủi ro* | Giá trị mang lại có vượt trội chi phí & rủi ro? | Nếu không → Đặt chốt chặn phê duyệt (HITL) hoặc chọn Not Yet / No-Go |

### 13.8 Cây quyết định Rule / Workflow / Agent

```
                 Logic / Input ổn định
                       │
        ┌──────────────┴──────────────┐
       Có                            Không
        │                             │
   ┌────┴────┐                ┌───────┴───────┐
  Cần đo               Cần plan          Cần nhiều tool
chất lượng?           dài/đa bước?       + ra quyết định?
   │                       │                    │
   ▼                       ▼                    ▼
 RULE                  WORKFLOW              AGENT
 + Eval/Check          (sequenced LLM)       (autonomous loop)
```

### 13.9 Ví dụ thực tế ngoài lớp học

| Chăm sóc khách hàng | Nghiên cứu bán hàng | Kho tri thức nội bộ |
|---------------------|---------------------|---------------------|
| **Rule**: Đọc luật phiếu, hỗ trợ đặt câu chuyện | **Rule**: Lọc khách hàng tiềm năng theo thuộc tính cố định | **Rule**: Phân quyền truy cập theo nhóm/danh mục |
| **Workflow**: Tóm tắt yêu cầu, kèm chú thích sản phẩm trước khi chuyển TA | **Workflow**: Thu thập thông tin từ Tìm dữ + Soạn email cộng tác | **Workflow**: Tra cứu chính sách, định hình bản tóm tắt câu trả lời |
| **Agent**: Tự ra quyết định trong các bước, tạo yêu cầu cộng tác | **Agent**: Giám sát hiệu chỉnh hành trình cộng tác, mỗi CRM, đề xuất kế hoạch | **Agent**: Phân tích chính sách mới, đề xuất bản tin nội bộ và tổng hợp hành chính nguồn dữ liệu |

### 13.10 Thiết kế UX và Human-in-the-loop (HITL)

| Nguyên tắc | Mô tả |
|------------|-------|
| **Làm rõ ý định** | Yêu cầu bổ sung ngữ cảnh hoặc làm rõ khi thông tin chưa đủ |
| **Minh bạch thông tin** | Trích dẫn nguồn cụ thể, minh chứng cho câu trả lời |
| **Phê duyệt thủ công** | Con người kiểm duyệt trước khi thực hiện tác vụ rủi ro cao |
| **Thiết lập ranh giới** | Giới hạn phạm vi và hoạt động tự chủ của AI để tránh hành vi ngoài kiểm soát |

> Dù mô hình tối ưu, **thiết kế UX không phù hợp** vẫn dẫn đến trải nghiệm người dùng kém hiệu quả.

---

## 14. Problem Statement hoàn chỉnh & Khung quyết định

Liên kết chặt chẽ giữa **bài toán, workflow, metrics và quyết định AI** — thành đầu vào cho Eval Plan.

### 14.1 Problem Statement cho hệ thống AI — 6 yếu tố bài toán cốt lõi + 3 yếu tố quyết định AI

| Trường | Ý nghĩa |
|--------|---------|
| **Actor** *(đối tượng ảnh hưởng)* | Đối tượng trực tiếp chịu ảnh hưởng bởi vấn đề |
| **Workflow** *(quy trình hiện tại)* | Quy trình vận hành hiện tại gồm các bước cụ thể thế nào? |
| **Bottleneck** *(nút thắt)* | Khâu nào gặp tình trạng chậm trễ, sai sót, lặp lại? |
| **Impact** *(tác động)* | Tổn thất lượng hóa bằng thời gian, chi phí, SLA hoặc chất lượng |
| **Success Metric** *(chỉ số thành công)* | Chỉ số đo lường cụ thể để xác định sự cải thiện |
| **Boundary** *(ranh giới)* | AI không được làm gì; khâu nào bắt buộc có con người |
| **Điểm AI can thiệp** *(decision · entry)* | AI hỗ trợ hoặc tự động hóa ở bước cụ thể thế nào? |
| **Mức chọn** *(decision · level)* | Rule / Workflow / Agent? |
| **Rủi ro & HITL** *(decision · safety)* | Phương án xử lý khi AI sai sót và quy trình phê duyệt thủ công |

### 14.2 Ví dụ mẫu — Hỗ trợ Lab Coach / TA

| Trường | Nội dung |
|--------|----------|
| **Actor** | Lab Coach hỗ trợ các nhóm học viên trong lớp 500 người |
| **Workflow** | Học viên đặt câu hỏi → Lab Coach nghiên cứu ngữ cảnh → Phản hồi / yêu cầu làm rõ → Học viên cập nhật |
| **Bottleneck** | Câu hỏi trùng lặp hoặc thiếu thông tin liên tục tăng cao; Lab Coach mất thời gian phân loại thủ công |
| **Impact** | Học viên chờ phản hồi lâu; Lab Coach quá tải, thiếu thời gian cho câu hỏi phức tạp |
| **Success Metric** | Giảm tỷ lệ câu hỏi lặp/duyệt thủ công; rút ngắn thời gian phản hồi trung bình, không tăng tỷ lệ định hướng sai |
| **Boundary** | AI không tự đánh giá/chấm điểm bài; chỉ hỗ trợ gợi ý và rà điều phối quy trình |
| **Điểm AI can thiệp** | Ngay sau khi học viên gửi câu hỏi hoặc Problem Card thiếu thông tin ngữ cảnh |
| **Mức chọn** | **Workflow** — AI phát hiện thông tin còn thiếu; Lab Coach phê duyệt câu hỏi chuyên sâu |
| **Rủi ro & HITL** | AI định hướng sai → Lab Coach kiểm duyệt trước khi gửi phản hồi |

### 14.3 Từ Problem Statement đến Eval Plan

Problem Statement rõ ràng giúp dễ kiểm thử:

| 01 — Input | 02 — Test Cases | 03 — Success |
|------------|-----------------|--------------|
| **Problem Statement** | **Kịch bản kiểm thử** | **Chỉ số hiệu năng** |
| 6 trường thông tin chính | Đa dạng tình huống thực tế (Edge Cases) | Chỉ số đo lường hiệu quả định lượng |
| **Tác vụ ưu tiên** | **Hiệu năng quy trình** | **Boundary & Sai sót** |
| Hệ thống có cần phải đo, ưu tiên tác vụ tự động? | Khi nào hệ thống bị sai và mức trễ là bao nhiêu? | Hệ thống xử lý ra sao khi gặp tình huống ngoài phạm vi (Court Boundary)? |

### 14.4 Khung ra quyết định: Go / Not Yet / No-Go

| **Go** *(thực hiện)* | **Not Yet** *(tạm hoãn)* | **No-Go** *(không triển khai)* |
|----------------------|---------------------------|--------------------------------|
| **Đủ điều kiện**: | **Có triển vọng**: | **Không phù hợp**: |
| Bài toán rõ ràng | Cần bổ sung dữ liệu thực tế | AI không mang giá trị vượt trội |
| Chỉ số đo lường khả thi | Chuẩn hóa quy trình | Rủi ro vận hành quá cao |
| Điểm can thiệp AI phù hợp | Thiết lập chỉ số | Giải pháp không dùng AI tối ưu hơn |
| Kiểm soát được rủi ro | Xác định ranh giới | |

> Quyết định **"Not Yet"** thể hiện sự chín chắn trong tư duy thiết kế sản phẩm, **không phải sự thất bại**.

---

## 15. Năm nguyên tắc cốt lõi sau Day 02

> Kim chỉ nam để thẩm định mọi đề xuất ứng dụng AI.

1. **Brief mơ hồ không thay thế Problem Statement.** Một bản tóm tắt mơ hồ không thay thế cho một Problem Statement hoàn chỉnh.
2. **Mô hình hóa workflow trước khi tích hợp AI.** Bắt buộc phải mô hình hóa quy trình trước khi xem xét tích hợp giải pháp AI.
3. **Pain point phải được lượng hóa.** Mọi điểm đau cần được lượng hóa bằng baseline và chỉ số đo lường cụ thể.
4. **Phức tạp không đồng nghĩa với hiệu quả.** Rule, Workflow và Agent là ba cấp độ khác nhau; độ phức tạp kỹ thuật không đồng nghĩa với hiệu quả tối ưu.
5. **Quyết định dựa trên lập luận thực tế.** Quyết định Go / Not Yet / No-Go phải được thiết lập dựa trên lập luận thực tế và số liệu kiểm thử rõ ràng.

---

## 16. Tài liệu tham khảo

<div class="slide-deck" id="day2-ref-deck">
  <div class="slide-stage">
    <img class="slide-img" id="day2-ref-img" src="Vin%20AI/Day-2/Reference/Screenshot_288.png" alt="Reference 1" />
  </div>
  <div class="slide-controls">
    <button type="button" class="slide-btn" id="day2-ref-prev" aria-label="Tham khảo trước">← Trước</button>
    <span class="slide-counter" id="day2-ref-counter">1 / 8</span>
    <button type="button" class="slide-btn" id="day2-ref-next" aria-label="Tham khảo kế">Sau →</button>
  </div>
  <div class="slide-hint">Bộ ảnh tham khảo bổ sung cho Day 2.</div>
</div>

<script>
(function () {
  var total = 8;
  var startNum = 288;
  var basePath = 'Vin%20AI/Day-2/Reference/Screenshot_';
  var current = 0;
  var img = document.getElementById('day2-ref-img');
  var counter = document.getElementById('day2-ref-counter');
  var prevBtn = document.getElementById('day2-ref-prev');
  var nextBtn = document.getElementById('day2-ref-next');
  if (!img || !counter || !prevBtn || !nextBtn) return;

  function render() {
    img.src = basePath + (startNum + current) + '.png';
    img.alt = 'Reference ' + (current + 1);
    counter.textContent = (current + 1) + ' / ' + total;
  }
  function prev() { current = (current - 1 + total) % total; render(); }
  function next() { current = (current + 1) % total; render(); }

  prevBtn.addEventListener('click', prev);
  nextBtn.addEventListener('click', next);

  render();
})();
</script>

---

> Hết Day 2. Mai sẽ học **ReAct pattern** — kiến trúc Agent cơ bản nhất.
