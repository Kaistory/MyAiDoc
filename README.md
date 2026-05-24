# My AI Doc

> Sổ tay học tập cá nhân — ghi chú, kiến thức, và tài liệu tham khảo.

Đây là không gian cá nhân nơi mình lưu trữ tài liệu, ghi chú, và những gì học được trong quá trình tự học. Site được xây dựng bằng [Docsify](https://docsify.js.org/) — chạy hoàn toàn tĩnh, không cần build step, không phụ thuộc Node.

## Nội dung chính

- **[Lộ trình Vin 15 ngày](docs/vin-15-ngay/attendance.md)** — học AI/LLM, Agent, RAG, LLMOps từ nền tảng tới production
- **[Lập trình](docs/lap-trinh/vi-du-ghi-chu.md)** — ngôn ngữ, framework, kỹ thuật code
- **[Nhật ký học tập](blog/2026-05-17-bat-dau.md)** — what I learned today
- **[Roadmap Firmware Engineer](roadmap.html)** — lộ trình middle dev

## Ba đặc điểm

**📝 Viết bằng Markdown** — Mọi ghi chú đều là file `.md`. Đơn giản, di động, version control sẵn qua git.

**🗂️ Tổ chức theo chủ đề** — Cấu trúc thư mục chính là cấu trúc sidebar. Thêm file mới = chỉnh `_sidebar.md`.

**🚀 Không cần cài gì** — Không Node, không build. Push lên GitHub Pages là chạy. Edit local chỉ cần text editor + browser.

## Cách thêm tài liệu mới

1. Tạo file `docs/<category>/<ten-bai>.md`
2. Thêm link vào `_sidebar.md`
3. `git push` → site live ngay

## Tech stack

- **[Docsify](https://docsify.js.org/)** — load markdown ở client, không build
- **GitHub Pages** — host miễn phí
- **GitHub Actions** — deploy tự động (chỉ upload, không build)
