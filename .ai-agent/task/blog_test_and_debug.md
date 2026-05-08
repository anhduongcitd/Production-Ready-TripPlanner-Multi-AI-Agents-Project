# Bài 8 — Test & Debug Agents (dành cho Fresher)

Mục tiêu: Hướng dẫn chiến lược test và debug cho agents, tools và pipeline trong repo TripPlanner — từ unit test nhỏ cho tool đến integration/e2e test cho toàn bộ flow.

---

## 1) Phân loại test

- Unit tests: test function nhỏ, tool wrapper, xử lý string, parsing.
- Integration tests: test agent dùng nhiều component (agent + tool) với external calls mocked.
- End‑to‑end (E2E): chạy toàn bộ app (ví dụ `demo.py`) với các key sandbox hoặc mock server.

## 2) Nguyên tắc chung

- Mock mọi external dependency (HTTP/API/Vector DB) để test nhanh, đáng tin cậy.
- Giữ output deterministic trong test bằng cách stub/patch trả giá trị cố định.
- Kiểm tra cả happy path và edge cases (empty response, API error, malformed data).

## 3) Unit test cho `tools`

Ví dụ test đơn giản cho `SearchFlights.search_flights_tool`:

```python
# tests/test_search_flights.py
from src.agentic.tools.search_flights import SearchFlights
from unittest.mock import patch

@patch('taskflowai.AmadeusTools.search_flights')
def test_search_flights(mock_amadeus):
    mock_amadeus.return_value = [{"flight": "ABC123", "price": 120}]
    fn = SearchFlights.search_flights_tool()
    res = fn({"from": "HAN", "to": "BKK"})
    assert isinstance(res, list)
    assert res[0]['price'] == 120
```

- Mục tiêu: đảm bảo wrapper gọi đúng API client và xử lý lỗi/định dạng đúng.

## 4) Unit test cho `agents` (mock LLM/Task)

- Nếu agent dùng `Task.create` để gửi job lên `taskflowai`, mock `Task.create` để trả nội dung cố định.

```python
# tests/test_reporter_agent.py
from src.agentic.agents.reporter_agent import TravelReportAgent
from unittest.mock import patch
from taskflowai import Task

@patch('taskflowai.Task.create')
def test_reporter_agent_create_report(mock_task_create):
    mock_task_create.return_value = '## Report\nContent'
    agent = TravelReportAgent.initialize_travel_report_agent()
    res = Task.create(agent=agent, context='Destination Report: ...', instruction='Create short report')
    assert 'Report' in res
```

- Kiểm tra: agent khởi tạo thành công và pipeline gọi `Task.create` không lỗi.

## 5) Integration tests (mock external APIs)

- Tạo fixtures mock cho Serper, Amadeus, WikipediaTools.
- Chạy flow: `research_destination()` → `Task.create` với `web_research_agent` và assert output chứa sections/markdown/ảnh.

Ví dụ:

```python
# tests/test_integration_flow.py
from demo import research_destination
from unittest.mock import patch

@patch('src.agentic.tools.serper_search.WebTools.serper_search')
@patch('src.agentic.tools.search_articles.WikipediaTools.search_articles')
@patch('src.agentic.tools.search_images.WikipediaTools.search_images')
def test_research_destination_flow(mock_images, mock_articles, mock_serper):
    mock_serper.return_value = 'Serper result'
    mock_articles.return_value = '## Article\nContent'
    mock_images.return_value = '![img](https://example.com/img.jpg)'

    task = research_destination('Bangkok', 'food')
    assert task is not None
```

## 6) E2E / Smoke tests

- Run `streamlit` in CI using small sandbox keys or mock server that simulates API behavior.
- Smoke test user flows (fill form → click plan → expect final markdown/download button).

## 7) Debugging tips

- Bật verbose: `set_verbosity(True)` (used in repo) để xem request/response.
- Log inputs/outputs ở mức debug inside tools.
- Replicate issue locally with a minimal script that calls the failing component.
- Use `print()` or logging to capture intermediate values (context, docs returned, prompt sent to LLM).

## 8) Common failure modes & fixes

- Missing env vars: `main_utils.py` checks `OPENAI_API_KEY`; `demo.py` lists other keys — ensure `.env` has them.
- API rate limit: mock in tests; add retry/backoff in production wrapper.
- Images not rendering: ensure markdown uses full `https://` URLs (helper `format_markdown_images` in `demo.py`).
- Long context exceed tokens: implement chunking and reduce k in retrieval.

## 9) CI & commands

- Requirements already in `requirements.txt`. Example CI steps:

```yaml
# .github/workflows/ci.yml (sketch)
steps:
  - uses: actions/checkout@v3
  - name: Set up Python
    uses: actions/setup-python@v4
    with:
      python-version: '3.10'
  - name: Install deps
    run: pip install -r requirements.txt
  - name: Run tests
    run: pytest -q
```

- Local run tests:

```bash
pip install -r requirements.txt
pytest -q
```

## 10) Kịch bản sửa lỗi nhanh

- Lỗi LLM unexpected: mock LLM and test prompt only.
- Lỗi tool timeout: increase timeout, add retry and circuit breaker.
- Lỗi mismatch giữa agent và tool contract: add schema validation (pydantic) in tool output.

---

Nếu bạn muốn, tôi có thể:
- Thêm một tập `tests/` mẫu (3–4 file) vào repo để bạn chạy ngay.  
- Hoặc tiếp tục viết `Triển khai Docker` bài tiếp theo.
