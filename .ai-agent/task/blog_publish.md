# Bài 10 — Tổng duyệt & Xuất bản (dành cho Fresher)

Mục tiêu: Danh sách kiểm tra cuối cùng trước khi xuất bản các bài blog kỹ thuật và hướng dẫn cách đưa nội dung lên repo/website/gh-pages hoặc nền tảng public.

---

## 1) Checklist nội dung

- Đảm bảo văn phong dễ hiểu, không lỗi chính tả.
- Code samples đã chạy được (thử local hoặc CI).
- Đã dẫn link đến các file tham chiếu trong repo (ví dụ: `src/agentic/agents/travel_agent.py`).
- Ảnh/asset có URL hoạt động hoặc đã tải lên CDN/GitHub và không chứa dữ liệu nhạy cảm.

## 2) Kiểm tra kỹ thuật trước khi publish

- Chạy unit tests và integration tests:

```bash
pip install -r requirements.txt
pytest -q
```

- Chạy demo hoặc Docker locally:

```bash
# streamlit local
streamlit run demo.py

# hoặc chạy Docker (nếu đã build image)
docker build -t tripplanner:latest .
docker run -p 8501:8501 -e OPENAI_API_KEY="${OPENAI_API_KEY}" tripplanner:latest
```

- Format & lint code (optional):

```bash
black src/ tests/
flake8 src/ --max-line-length=120
```

## 3) Phiên bản, commit và PR

- Tạo branch riêng cho bài viết:

```bash
git checkout -b docs/blog/agentic-series
```

- Thêm file bài viết trong `.ai-agent/task/` hoặc trong `docs/` (tuỳ chiến lược). Commit & push:

```bash
git add .ai-agent/task/blog_*.md
git commit -m "docs: add Agentic blog series (Docker + Publish)"
git push origin HEAD
```

- Tạo Pull Request, yêu cầu review.

## 4) Đưa blog lên GitHub Pages / docs

A) Dùng `docs/` folder: đặt các markdown vào `docs/` và bật GitHub Pages từ `main` branch -> `docs/` folder.

B) Dùng static site generator (MkDocs/Jekyll):
- Tạo site bằng MkDocs, copy markdown vào `docs/` hoặc `site` theme, cấu hình deploy bằng GitHub Actions.

## 5) Xuất bản lên nền tảng public (Medium / Dev.to)

- Chuẩn hoá markdown: đảm bảo đường dẫn ảnh đầy đủ và code block có tag ngôn ngữ.
- Tạo một post mới trên Medium/Dev.to, paste nội dung (hoặc dùng API để tự động upload).
- Đính kèm nguồn gốc (link repo) và license nếu cần.

## 6) Post‑publish: kiểm tra & quảng bá

- Kiểm tra hiển thị: open post trên nhiều thiết bị (mobile/desktop).
- Share link lên Slack/Twitter/LinkedIn hoặc kênh nội bộ.
- Thêm vào `README.md` hoặc `docs/index.md` liên kết đến loạt bài.

## 7) Gợi ý workflow CI cho blog (gợi ý)

- Tạo GitHub Action: khi push branch `docs/*` hoặc khi merge PR vào `main`, chạy `pytest`, build tài liệu (MkDocs) và deploy lên `gh-pages` branch.

## 8) Kết luận ngắn

- Luôn test code snippets và demo trước khi publish.
- Không để lộ API keys hoặc thông tin nhạy cảm trong bài viết.
- Gắn nguồn code trong repo để người đọc dễ tìm hiểu tiếp.

---

Muốn tôi:  
- Tạo `docker-compose.yml` mẫu trong repo?  
- Hoặc commit 2 bài blog này vào branch và mở PR giúp bạn?