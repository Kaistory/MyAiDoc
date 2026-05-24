---
title: Giới thiệu
---

# Chào mừng đến với My AI Doc

Đây là không gian cá nhân nơi mình lưu trữ tài liệu, ghi chú, và những gì học được trong quá trình tự học. Site được xây dựng bằng [Docsify](https://docsify.js.org/) — chạy hoàn toàn tĩnh, không cần build step, không phụ thuộc Node.

## Cấu trúc

Tài liệu được tổ chức theo chủ đề:

- **Vin 15 ngày** — lộ trình học AI/LLM, Agent, RAG, LLMOps từ nền tảng tới production
- **Lập trình** — note về ngôn ngữ, framework, kỹ thuật code
- **AI & Machine Learning** — khái niệm, mô hình, paper
- **Công cụ** — CLI, editor, workflow
- **Ghi chú chung** — những điều linh tinh khác

## Cách thêm tài liệu mới

Mỗi tài liệu là một file `.md` trong thư mục `docs/`. Sau khi tạo file, thêm link vào `_sidebar.md` để hiện trong sidebar.

Ví dụ: tạo file `docs/lap-trinh/javascript-closure.md` với nội dung:

```markdown
---
title: JavaScript Closure
---

# JavaScript Closure

Closure là...
```

Sau đó thêm dòng vào `_sidebar.md`:

```markdown
- [JavaScript Closure](docs/lap-trinh/javascript-closure.md)
```

`git push` lên GitHub, GitHub Actions sẽ tự deploy.

## Chạy local

Không cần install gì. Có thể dùng một trong các cách:

- **VS Code** — cài extension *Live Preview* và mở `index.html`
- **Python** (có sẵn trên hầu hết máy): `python -m http.server 3000` rồi mở `http://localhost:3000`
- **Chỉ cần edit** rồi `git push` — xem live trên GitHub Pages
