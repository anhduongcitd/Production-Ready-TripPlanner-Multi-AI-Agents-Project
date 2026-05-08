# Bài 4 — Triển khai `ReporterAgent` (dành cho Fresher)

Mục tiêu: Giải thích vai trò `ReporterAgent` trong repo TripPlanner, cách khởi tạo, cách dùng để tổng hợp báo cáo cuối cùng (giữ markdown và ảnh), và cách mở rộng.

---

## 1) Mục đích của `ReporterAgent`

- `ReporterAgent` chịu trách nhiệm nhận các phần nội dung đã thu thập (destination, events, weather, flights) và tổng hợp thành báo cáo hoàn chỉnh.
- Nhiệm vụ chính: format, giữ nguyên markdown (đặc biệt là ảnh), thêm tiêu đề/cấu trúc, và xuất ra định dạng người dùng đọc/ tải về.

## 2) Vị trí file và mã nguồn chính

- File agent: `src/agentic/agents/reporter_agent.py`

Đoạn mã khởi tạo trong repo:

```python
from taskflowai import Agent
from src.agentic.utils.main_utils import LoadModel
from src.agentic.logger import logging
from src.agentic.exception import CustomException
import sys

class TravelReportAgent:
    @classmethod
    def initialize_travel_report_agent(cls):
        try:
            logging.info("Initializing the Travel Report Agent.")

            travel_report_agent = Agent(
                role="Travel Report Agent",
                goal="Write comprehensive travel reports with visual elements",
                attributes="friendly, hardworking, visual-oriented, and detailed in reporting",
                llm=LoadModel.load_openai_model()
            )

            logging.info("Travel Report Agent initialized successfully.")
            return travel_report_agent

        except Exception as e:
            logging.info("An unexpected error occurred")
            raise CustomException(sys, e)
```

Ghi chú: `reporter_agent` thường chỉ cần `llm` (không cần tools) vì nhiệm vụ chính là tổng hợp và format.

## 3) Ví dụ sử dụng trong `demo.py`

Trong `demo.py`, hàm `write_travel_report(...)` dùng `Task.create(...)` với `reporter_agent` để tạo báo cáo cuối cùng:

```python
def write_travel_report(destination_report, events_report, weather_report, flight_report):
    instruction = (
        "Create a comprehensive travel report that:\n"
        "1. Maintains all images from the destination and events reports\n"
        "2. Organizes information in a clear, logical structure\n"
        "3. Keeps all markdown formatting intact\n"
        "4. Ensures images are properly displayed with captions\n"
        "5. Includes all key information from each section"
    )
    return Task.create(
        agent=reporter_agent,
        context=f"Destination Report: {destination_report}\n\n"
               f"Events Report: {events_report}\n\n"
               f"Weather Report: {weather_report}\n\n"
               f"Flight Report: {flight_report}",
        instruction=instruction
    )
```

- Kết quả trả về thường là markdown/string đã format — `demo.py` dùng `format_markdown_images()` để đảm bảo URL ảnh đầy đủ và `st.markdown()` để hiển thị.

## 4) Thiết kế & best practices cho ReporterAgent

- Giữ instruction rõ ràng: nêu rõ cần giữ markdown/ảnh, cấu trúc đầu mục, và giới hạn chiều dài nếu cần.
- Truyền `context` có cấu trúc (label/section) để LLM biết nguồn của từng phần.
- Nếu cần visual nâng cao (bảng/biểu đồ), có 2 hướng:
  - LLM trả markdown + placeholder cho ảnh/biểu đồ, sau đó pipeline chèn image/chart thật.
  - ReporterAgent gọi một utility (tool) nội bộ để vẽ biểu đồ (matplotlib/plotly) và trả link ảnh.

## 5) Xử lý ảnh và markdown trong repo

- `demo.py` có hàm `format_markdown_images()` để đảm bảo tất cả ảnh có `http://` hoặc `https://` trước khi render.
- Kỹ thuật hay dùng: yêu cầu LLM trả hình ảnh dưới dạng markdown `![Alt text](https://full-url)` để renderer (Streamlit, markdown) hiển thị tự động.

## 6) Mở rộng ReporterAgent (ý tưởng thực tế)

- Template system: dùng Jinja2 để parse markdown sections và render PDF/HTML với CSS.
- Xuất PDF: render markdown → HTML → convert to PDF (weasyprint, wkhtmltopdf).
- Thêm gallery ảnh: Lưu ảnh cục bộ hoặc upload lên CDN và chèn URL vào markdown.
- Tạo CV/one-pager tóm tắt (short summary + highlights).

## 7) Kiểm thử nhanh (mock)

Ví dụ test đơn giản với `pytest` bằng cách mock `Task.create`:

```python
# tests/test_reporter_agent.py
from src.agentic.agents.reporter_agent import TravelReportAgent
from unittest.mock import patch
from taskflowai import Task

def test_reporter_agent_create_report():
    agent = TravelReportAgent.initialize_travel_report_agent()
    sample_context = (
        "Destination Report: ## Place\nSome content\n"
        "Events Report: ## Events\nSome content\n"
    )
    instruction = "Create concise report"

    with patch('taskflowai.Task.create') as mock_create:
        mock_create.return_value = "## Final Report\nSample content with ![img](https://example.com/img.jpg)"
        result = Task.create(agent=agent, context=sample_context, instruction=instruction)
        assert "Final Report" in result
        assert "![](" in result or "!" in result
```

- Trong test thực tế, mock tool/LLM để tránh gọi mạng.

## 8) Chạy thử và kiểm tra

Cài dependencies và chạy Streamlit demo:

```bash
pip install -r requirements.txt
streamlit run demo.py
```

Hoặc gọi thử reporter agent trong script nhanh:

```bash
python -c "from src.agentic.agents.reporter_agent import TravelReportAgent; from taskflowai import Task; a=TravelReportAgent.initialize_travel_report_agent(); print(Task.create(agent=a, context='Destination Report: ...', instruction='Create short report'))"
```

## 9) Tóm tắt

- `ReporterAgent` là bước cuối cùng của pipeline: tổng hợp, format và trả về báo cáo cho người dùng.
- Giữ context có cấu trúc và instruction chính xác giúp LLM thực hiện tốt nhiệm vụ.
- Test bằng mock để đảm bảo workflow hoạt động mà không phụ thuộc mạng.

---

Tiếp theo: tôi sẽ viết bài `WebResearchAgent & Serper` (tích hợp Serper & Wikipedia) — tiếp tục chứ?