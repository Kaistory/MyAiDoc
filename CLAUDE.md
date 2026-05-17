# MyAiDoc

Site cá nhân lưu tài liệu học tập của Kaistory. Lấy cảm hứng từ [d2l.ai](https://d2l.ai/).

## Stack
- **Docusaurus 3** (React + Node) — static site generator
- **TypeScript** cho config & components
- **MDX** cho nội dung (markdown + React component)
- **GitHub Pages** để host (free), deploy qua **GitHub Actions** trên push tới `main`

## URL
- Production: https://kaistory.github.io/MyAiDoc/
- Dev: http://localhost:3000/MyAiDoc/

## Cấu trúc

```
docs/                       # nội dung tài liệu (markdown/mdx)
  intro.mdx                 # trang giới thiệu
  lap-trinh/                # category — ghi chú lập trình
  ai-ml/                    # category — AI/ML
  cong-cu/                  # category — công cụ
  ghi-chu/                  # category — ghi chú chung
  <category>/_category_.json # metadata cho category (label, position)
blog/                       # nhật ký học tập
  authors.yml               # thông tin tác giả
src/
  pages/index.tsx           # trang chủ
  components/HomepageFeatures/  # các block trên trang chủ
  css/custom.css            # CSS toàn site
static/img/                 # ảnh, logo, favicon
docusaurus.config.ts        # config chính
sidebars.ts                 # autogenerate từ docs/
.github/workflows/deploy.yml  # CI deploy GitHub Pages
```

## Lệnh thường dùng

```bash
npm start          # dev server, http://localhost:3000/MyAiDoc/
npm run build      # build production vào build/
npm run serve      # preview build/ tại localhost:3000
npm run clear      # xóa cache .docusaurus/
npm run typecheck  # kiểm tra TypeScript
```

## Thêm tài liệu mới

1. Tạo file `docs/<category>/<ten-bai>.md` (hoặc `.mdx` nếu cần React component)
2. Thêm frontmatter:
   ```yaml
   ---
   title: Tiêu đề bài
   sidebar_position: 1  # optional, để thứ tự
   ---
   ```
3. `git push` → site tự deploy.

## Thêm category mới

1. Tạo thư mục `docs/<ten-category>/`
2. Tạo `docs/<ten-category>/_category_.json` với label, position, description
3. Thêm các file `.md` bên trong

## Deploy

Tự động qua `.github/workflows/deploy.yml` khi push lên `main`. Cần bật GitHub Pages trong repo settings:
**Settings → Pages → Source: GitHub Actions**.

## Quy ước

- Ngôn ngữ chính: **tiếng Việt** (`defaultLocale: 'vi'` trong config)
- Đặt tên file: kebab-case không dấu (`closure-trong-javascript.md`)
- Hình ảnh để trong `static/img/<category>/`
- Code block luôn kèm language tag: ` ```js `, ` ```python `, ` ```bash `
