# MyAiDoc

Site cá nhân lưu tài liệu học tập của Kaistory. Lấy cảm hứng từ [d2l.ai](https://d2l.ai/).

## Stack

- **[Docsify](https://docsify.js.org/)** — load markdown ở client, **không build step**, **không Node**
- Markdown thuần `.md` cho mọi nội dung
- **GitHub Pages** host miễn phí
- **GitHub Actions** chỉ upload thư mục — không cần install gì

## URL

- Production: https://kaistory.github.io/MyAiDoc/
- Local: mở `index.html` qua VS Code Live Preview, hoặc `python -m http.server 3000`

## Cấu trúc

```
/
├── index.html              # Docsify entry (load Docsify JS từ CDN)
├── README.md               # Trang chủ (#/)
├── _sidebar.md             # Sidebar nav (toàn site)
├── _navbar.md              # Top navbar
├── _coverpage.md           # Cover page
├── .nojekyll               # Cho GitHub Pages biết bỏ qua Jekyll
├── docs/                   # Tài liệu
│   ├── intro.md
│   ├── vin-15-ngay/        # Lộ trình 15 ngày
│   └── lap-trinh/          # Note lập trình
├── blog/                   # Nhật ký
│   └── 2026-05-17-bat-dau.md
├── img/                    # Ảnh, logo, favicon
├── roadmap.html            # Trang HTML tĩnh độc lập
└── .github/workflows/
    └── deploy.yml          # Upload + deploy GitHub Pages
```

## Chạy local

Không cần install gì:

```bash
# Cách 1: Python (có sẵn trên hầu hết máy)
python -m http.server 3000
# rồi mở http://localhost:3000

# Cách 2: VS Code — cài extension "Live Preview", mở index.html

# Cách 3: Push lên GitHub Pages, xem trực tiếp
```

## Thêm tài liệu mới

1. Tạo `docs/<category>/<ten-bai>.md` với frontmatter:
   ```yaml
   ---
   title: Tiêu đề bài
   ---
   ```
2. Thêm dòng tương ứng vào `_sidebar.md`:
   ```markdown
   - [Tiêu đề bài](docs/<category>/<ten-bai>.md)
   ```
3. `git push` → site live ngay.

## Thêm category mới

1. Tạo thư mục `docs/<ten-category>/`
2. Thêm section mới vào `_sidebar.md`:
   ```markdown
   - **Tên category**
     - [Bài đầu tiên](docs/<ten-category>/bai-1.md)
   ```
3. Thêm các file `.md` vào thư mục

## Deploy

Tự động qua `.github/workflows/deploy.yml` khi push lên `main`. Cần bật GitHub Pages trong repo:
**Settings → Pages → Source: GitHub Actions**.

Workflow chỉ upload toàn bộ thư mục lên GitHub Pages — **không có build step**, **không install Node**.

## Quy ước

- Ngôn ngữ chính: **tiếng Việt**
- Đặt tên file: kebab-case không dấu (`closure-trong-javascript.md`)
- Hình ảnh để trong `img/<category>/`
- Code block luôn kèm language tag: ` ```js `, ` ```python `, ` ```bash `
- Admonitions: dùng blockquote chuẩn, ví dụ:
  ```markdown
  > 💡 **Mẹo — Title**
  >
  > Nội dung mẹo
  ```

## Lưu ý về Docsify

- File markdown được fetch + render ở **client**, KHÔNG có pre-build HTML
- Mọi link tới file `.md` trong sidebar phải có đuôi `.md`
- Frontmatter (`---`) Docsify không render nhưng cũng không hiện ra, vẫn dùng được
- Không hỗ trợ MDX (React component trong markdown) — chỉ markdown thuần + HTML inline

## Migration history

Trước đây site dùng Docusaurus 3 (cần Node + npm). Đã migrate sang Docsify (2026-05-24) để loại bỏ phụ thuộc Node — code chạy được trên máy khác mà không cần cài gì. Lịch sử Docusaurus có thể xem trong git log (commit trước `9690bcf base-website`).
