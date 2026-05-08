# Bài 2 — Kiến trúc agent (dành cho Fresher)

Mục tiêu: Giải thích kiến trúc tổng thể của hệ thống agent trong repo TripPlanner, chỉ ra các thành phần chính, luồng dữ liệu và ví dụ thực tế từ mã nguồn để người mới dễ nắm.

---

## 1) Thành phần chính

- LLM & Loader
  - `src/agentic/utils/main_utils.py` chứa `LoadModel.load_openai_model()` — nơi cấu hình model và kiểm tra biến môi trường.
- Agents
  - Các agent đặt ở `src/agentic/agents/` (ví dụ: `travel_agent.py`, `reporter_agent.py`, `web_research_agent.py`). Mỗi file định nghĩa hàm khởi tạo agent (ví dụ `initialize_travel_agent`).
- Tools (wrapper cho API bên ngoài)
  - `src/agentic/tools/` chứa các wrapper như `search_flights.py`, `serper_search.py`, `search_articles.py`, `get_weather_data.py`.
  - Mỗi tool trả về một callable hoặc hàm mà `Agent` có thể dùng.
- Utilities / Infra
  - Logging và CustomException: dùng để ghi log và bọc lỗi (xem `src/agentic/logger` và `src/agentic/exception`).
- Chạy & Demo
  - `demo.py` và notebook trong `notebooks/` cho ví dụ runtime và thử nghiệm.

## 2) Luồng dữ liệu (step-by-step)

1. Người dùng gửi yêu cầu (ví dụ: "Tìm chuyến bay từ HN sang BKK ngày 2026-06-15").
2. Yêu cầu được chuyển tới một Agent đã khởi tạo (ví dụ `TravelAgent`). Agent có:
   - `role`, `goal`, `attributes` — giúp định hướng prompt / hành vi.
   - `llm` — model được nạp từ `LoadModel` để thực hiện reasoning/generation.
   - `tools` — danh sách các tool callable (ví dụ `SearchFlights`, `GetWeatherData`).
3. Agent phân tích input (bằng LLM), quyết định gọi tool nào.
4. Tool wrapper gọi API bên ngoài (ví dụ Amadeus, Serper, Wikipedia), trả về dữ liệu có cấu trúc.
5. Agent dùng LLM để tổng hợp/đọc dữ liệu trả về và tạo câu trả lời cho người dùng.
6. (Tùy chọn) Nếu cần báo cáo/visual, chuyển kết quả sang `ReporterAgent` để format/visualize.

## 3) Ví dụ minh họa từ mã nguồn

- Khởi tạo `TravelAgent` (xem `src/agentic/agents/travel_agent.py`):

```python
from src.agentic.agents.travel_agent import TravelAgent

# Khởi tạo object agent (sẵn sàng gọi tools)
travel_agent = TravelAgent.initialize_travel_agent()
```

- Tool wrapper ví dụ (`src/agentic/tools/search_flights.py`):

```python
class SearchFlights:
    @classmethod
    def search_flights_tool(cls):
        search_flights = AmadeusTools.search_flights
        return search_flights
```

Ghi chú: mã nguồn tách rõ phần "agent" (chỉ định role, goal, llm, tools) và phần "tool" (giao tiếp với API). Điều này giúp test độc lập và dễ mở rộng.

## 4) Thiết kế quyết định (tại sao theo pattern này?)

- Tách `Agent` và `Tool` giúp:
  - Viết test cho từng tool mà không phụ thuộc LLM.
  - Dễ thêm tool mới mà không chỉnh logic agent.
  - Giữ prompt / behavior rõ ràng nhờ `role`/`goal`/`attributes`.
- `LoadModel` trung tâm giúp kiểm soát model và validate biến môi trường sớm.

## 5) Thực hành: thêm tool mới (quick checklist)

1. Tạo file mới `src/agentic/tools/my_tool.py`.
2. Viết wrapper class với phương thức trả callable (theo mẫu `SearchFlights`).
3. Import và thêm vào danh sách `tools` khi khởi tạo agent (ví dụ `TravelAgent`).
4. Viết unit test cho wrapper tool, mock API ngoại vi.

Ví dụ template:

```python
# src/agentic/tools/my_tool.py
class MyTool:
    @classmethod
    def my_callable(cls):
        def call(params):
            # gọi API hoặc xử lý
            return {"result": ...}
        return call
```

Và trong agent:

```python
from src.agentic.tools.my_tool import MyTool
# ... tools=[..., MyTool.my_callable()]
```

## 6) Best practices ngắn gọn

- Giữ output của tool có cấu trúc (dict/JSON) để LLM dễ tiêu hóa.
- Log đầy đủ input/output ở mức debug (thông tin nhạy cảm cần mask).
- Validate biến môi trường và key sớm (xem `main_utils.py`).
- Viết test cho từng tool, không để test phụ thuộc mạng thật.

---

Nếu bạn muốn, tôi sẽ tiếp tục:  
- Viết bài tiếp theo `Triển khai TravelAgent` (bắt đầu từ ví dụ cụ thể và testcase).  
- Hoặc chèn phần ví dụ chạy cụ thể dựa trên `demo.py`/notebook.

