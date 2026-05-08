# Bài 5 — WebResearchAgent & Serper (dành cho Fresher)

Mục tiêu: Giải thích vai trò của `WebResearchAgent` trong pipeline TripPlanner, cách agent tích hợp với Serper và WikipediaTools, cách viết instruction tốt, và cách test/mở rộng.

---

## 1) Vai trò chính

- `WebResearchAgent` chuyên tìm thông tin web và ảnh chất lượng cho điểm đến (destination) và sự kiện.
- Nó thường dùng các tool web (Serper) và công cụ Wikipedia để tìm bài viết và ảnh.

## 2) Các file liên quan

- Agent: `src/agentic/agents/web_research_agent.py`
- Tool wrappers: `src/agentic/tools/serper_search.py`, `src/agentic/tools/search_articles.py`, `src/agentic/tools/search_images.py`
- Demo sử dụng: `demo.py` (hàm `research_destination` và `research_events` gọi agent này)

## 3) Mã nguồn chính (tóm tắt)

Trong `src/agentic/agents/web_research_agent.py` agent được khởi tạo như sau:

```python
web_research_agent = Agent(
    role="Web Research Agent",
    goal="Research destinations and find relevant images",
    attributes="diligent, thorough, comprehensive, visual-focused",
    llm=LoadModel.load_openai_model(),
    tools=[SerperSearch.search_web(), WikiArticles.fetch_articles(), WikiImages.search_images()],
)
```

Mỗi phần tử trong `tools` là một callable trả về bởi một wrapper trong `src/agentic/tools/`.

## 4) Tool wrappers (nhìn vào code)

- `src/agentic/tools/serper_search.py` sử dụng `WebTools.serper_search`:

```python
class SerperSearch:
    @classmethod
    def search_web(cls):
        search = WebTools.serper_search
        return search
```

- `src/agentic/tools/search_articles.py` và `src/agentic/tools/search_images.py` dùng `WikipediaTools` tương tự.

Ý tưởng: wrapper tách logic gọi API ra khỏi agent, giúp test/mocking dễ dàng.

## 5) Làm sao agent được dùng trong `demo.py`

- `research_destination(destination, interests)` tạo `Task.create(...)` với `web_research_agent` và một `instruction` rất cụ thể (xem `demo.py`) — instruction yêu cầu:
  - 2-3 ảnh chất lượng, format ảnh là `![Alt](https://full-url)`
  - headings, captions, thông tin thực tế

Ví dụ instruction trích từ `demo.py`:

```
Create a comprehensive report about {destination} with the following:
1. Use Wikipedia tools to find and include 2-3 high-quality images of key attractions
2. Ensure images are full URLs starting with http:// or https://
3. Format images as: ![Description](https://full-image-url)
...
Format the entire response in clean markdown
```

Kết quả `Task.create` thường trả về markdown/string đã format; `demo.py` dùng `format_markdown_images()` để đảm bảo URL ảnh hợp lệ trước khi hiển thị.

## 6) Hướng dẫn viết instruction tốt cho WebResearchAgent

- Rõ ràng số lượng ảnh và bắt buộc URL đầy đủ (`http(s)://`).
- Yêu cầu caption cho mỗi ảnh.
- Yêu cầu markdown output nếu muốn hiển thị trực tiếp.
- Giới hạn chiều dài nếu cần (ví dụ "Give 3 short paragraphs per attraction").
- Gõ rõ các nguồn ưu tiên nếu muốn (ví dụ: "Use Wikipedia and official tourism sites").

## 7) Kiểm thử (unit & integration)

- Unit test: mock `WebTools.serper_search` và `WikipediaTools.*` để trả dữ liệu mẫu. Không gọi API thật.

Ví dụ pytest (đơn giản):

```python
from src.agentic.agents.web_research_agent import WebResearchAgent
from unittest.mock import patch

@patch('src.agentic.tools.serper_search.WebTools.serper_search')
@patch('src.agentic.tools.search_articles.WikipediaTools.search_articles')
@patch('src.agentic.tools.search_images.WikipediaTools.search_images')
def test_research_destination(mock_images, mock_articles, mock_serper):
    mock_serper.return_value = 'Serper sample result'
    mock_articles.return_value = '## Article sample\nContent'
    mock_images.return_value = '![img](https://example.com/img.jpg)'

    agent = WebResearchAgent.initialize_web_research_agent()
    from taskflowai import Task
    res = Task.create(agent=agent, context='Destination: X', instruction='Test')
    assert res is not None
```

- Integration: chạy `demo.py` locally with valid API keys and kiểm tra luồng full (nhưng nhớ giới hạn rate).

## 8) Lưu ý vận hành và lỗi thường gặp

- Missing API key: kiểm tra `SERPER_API_KEY` và các key khác trong `.env`.
- Rate limit / Quota: handle retry/backoff ở tool wrapper nếu API giới hạn.
- Ảnh không có đầy đủ URL: dùng `format_markdown_images()` (repo đã có helper này).
- Nội dung không phù hợp: thêm các constraint trong instruction ("Do not include adult content").

## 9) Mở rộng

- Thêm nguồn tìm kiếm khác (Bing, Google, Scraper) bằng cách viết wrapper mới trong `src/agentic/tools/` rồi thêm vào `tools` list.
- Cache kết quả tìm kiếm (Redis hoặc in-memory) để giảm phí và số gọi API.
- Tạo pipeline kiểm duyệt ảnh (kích thước, độ phân giải).

---

Nếu muốn, tôi sẽ tiếp tục viết bài `Tools & Utilities` ngay bây giờ (đã tạo file tương ứng).