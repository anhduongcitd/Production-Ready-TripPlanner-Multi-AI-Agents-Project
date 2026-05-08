# Bài 3 — Triển khai `TravelAgent` (dành cho Fresher)

Mục tiêu: Hướng dẫn chi tiết cách `TravelAgent` được triển khai trong repo, cách khởi tạo, cách gọi, và cách mở rộng — kèm ví dụ thực tế từ mã nguồn.

---

## 1) Tổng quan nhanh

- File chính: `src/agentic/agents/travel_agent.py` — nơi khởi tạo agent chuyên xử lý yêu cầu liên quan đến chuyến bay và thời tiết.
- Tools dùng: `SearchFlights` (Amadeus), `GetWeatherData` (weather API) — các wrapper nằm trong `src/agentic/tools/`.
- Model loader: `LoadModel.load_openai_model()` trong `src/agentic/utils/main_utils.py` trả model LLM đã cấu hình.

## 2) Mã nguồn chính (nhìn nhanh)

Đây là đoạn khởi tạo `TravelAgent` trong repo:

```python
from taskflowai import Agent, AmadeusTools, WebTools
from src.agentic.utils.main_utils import LoadModel
from src.agentic.logger import logging
from src.agentic.exception import CustomException
import sys
from src.agentic.tools.search_flights import SearchFlights
from src.agentic.tools.get_weather_data import GetWeatherData

class TravelAgent:
    @classmethod
    def initialize_travel_agent(cls):
        try:
            logging.info("Initializing the Travel Agent.")

            travel_agent = Agent(
                role="Travel Agent",
                goal="Assist travelers with their queries",
                attributes="friendly, hardworking, and detailed in reporting back to users",
                llm=LoadModel.load_openai_model(),
                tools=[SearchFlights.search_flights_tool(), GetWeatherData.fetch_weather_data()]
            )

            logging.info("Travel Agent initialized successfully.")
            return travel_agent

        except Exception as e:
            logging.info("An unexpected error occurred")
            raise CustomException(sys, e)
```

Giải thích ngắn:
- `Agent(...)`: định nghĩa role/goal/attributes để hướng prompt và hành vi của agent.
- `llm`: một model đã được load (ở đây là OpenAI GPT model theo `main_utils`).
- `tools`: danh sách các callable mà agent có thể gọi để lấy dữ liệu ngoài (flight, weather, v.v.).
- Error handling: bọc lỗi vào `CustomException` giúp trace và logging nhất quán.

## 3) Cách gọi `TravelAgent` (ví dụ chạy)

Repo sử dụng `Task.create(...)` từ `taskflowai` để gửi một nhiệm vụ tới agent và nhận trả về content (string/markdown). Ví dụ đơn giản:

```python
from src.agentic.agents.travel_agent import TravelAgent
from taskflowai import Task

travel_agent = TravelAgent.initialize_travel_agent()

# Tạo task tìm chuyến bay
flight_task = Task.create(
    agent=travel_agent,
    context="Flights from Hanoi to Bangkok on 2026-06-15",
    instruction="Find top 3 affordable flight options with departure/arrival times and price estimates"
)

print(flight_task)  # thường trả về markdown/string mà bạn có thể hiển thị
```

Trong `demo.py` (ứng dụng Streamlit), pattern tương tự được dùng: khởi tạo `travel_agent` và gọi `Task.create(agent=travel_agent, ...)` trong hàm `search_flights(...)`.

## 4) Thiết lập môi trường (env vars)

`main_utils.py` kiểm tra `OPENAI_API_KEY`. Ở `demo.py` còn liệt kê thêm các API keys được dùng bởi tools:

- `WEATHER_API_KEY`
- `SERPER_API_KEY`
- `AMADEUS_API_KEY` và `AMADEUS_API_SECRET`
- `GROQ_API_KEY`
- `GEMINI_API_KEY`
- `OPENAI_API_KEY`

Tạo file `.env` ở project root với các biến cần thiết hoặc export trong shell.

Ví dụ `.env`:

```
OPENAI_API_KEY=sk-...
AMADEUS_API_KEY=...
AMADEUS_API_SECRET=...
SERPER_API_KEY=...
WEATHER_API_KEY=...
```

## 5) Chạy bản demo (local)

Cài dependency và chạy Streamlit demo:

```bash
pip install -r requirements.txt
streamlit run demo.py
```

Hoặc nếu chỉ muốn test nhanh `TravelAgent` từ REPL/script:

```bash
python -c "from src.agentic.agents.travel_agent import TravelAgent; from taskflowai import Task; a=TravelAgent.initialize_travel_agent(); print(Task.create(agent=a, context='Hanoi->BKK 2026-06-15', instruction='Find flights'))"
```

## 6) Mở rộng `TravelAgent` — thêm tool mới

Quick checklist để thêm tool:

1. Tạo file wrapper trong `src/agentic/tools/`, ví dụ `my_tool.py`, theo pattern `class MyTool: @classmethod def my_callable(cls): return callable`.
2. Import wrapper vào `src/agentic/agents/travel_agent.py`.
3. Thêm `MyTool.my_callable()` vào danh sách `tools` khi khởi tạo agent.
4. Viết unit test cho wrapper (mock API) và integration test nhỏ cho task flow.

Ví dụ template tool:

```python
# src/agentic/tools/my_tool.py
class MyTool:
    @classmethod
    def my_callable(cls):
        def call(params):
            # gọi API hoặc xử lý nội bộ
            return {"result": "ok"}
        return call
```

Và thêm vào agent:

```python
from src.agentic.tools.my_tool import MyTool
# ... tools=[..., MyTool.my_callable()]
```

## 7) Test & Debug nhanh

- Bật verbose logging (repo đã gọi `set_verbosity(True)` trong nhiều chỗ) để xem request/response.  
- Log input/output của tool ở mức debug; mask keys khi cần.  
- Để test offline, mock các callable tool để trả dữ liệu mẫu.

Ví dụ test đơn giản với pytest (mock tool callable):

```python
# tests/test_travel_agent.py
from src.agentic.agents.travel_agent import TravelAgent
from unittest.mock import patch

def test_initialize_and_call():
    agent = TravelAgent.initialize_travel_agent()
    # Mock SearchFlights callable nếu cần ở mức integration
    # Tùy theo API của Task.create; kiểm tra output không ném exception
    from taskflowai import Task
    result = Task.create(agent=agent, context='Hanoi->BKK', instruction='Test')
    assert result is not None
```

(Trong thực tế bạn sẽ mock các external API để tránh gọi mạng thật.)

## 8) Một số lưu ý thiết kế

- Giữ `tools` trả output có cấu trúc (dict/JSON hoặc markdown) để LLM dễ tổng hợp.
- Tách rời `Agent` và `Tool` để test và bảo trì dễ dàng.
- Xử lý lỗi rõ ràng (CustomException) và log đầy đủ.

---

Nếu bạn muốn, tôi có thể tiếp tục và:
- Viết phần "Triển khai `ReporterAgent`" tiếp theo (chi tiết về cách format/visualize report).  
- Hoặc thêm ví dụ unit test mock cụ thể cho `SearchFlights`.

Bạn muốn tôi làm gì tiếp?