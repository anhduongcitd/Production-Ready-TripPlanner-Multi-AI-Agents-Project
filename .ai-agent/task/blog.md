# Bài 1 — Giới thiệu Agentic AI (dành cho Fresher)

Mục tiêu: Giải thích khái niệm "agentic" trong dự án TripPlanner, mô tả kiến trúc tổng quan, và dùng code trong repo làm ví dụ thực tế để người mới dễ hiểu.

---

## 1. Agent là gì (ở mức cơ bản)

- Agent = một thực thể phần mềm có "role" (vai trò), "goal" (mục tiêu), và khả năng gọi các "tools" (công cụ) để thực hiện nhiệm vụ.
- Trong repo này, agents được khởi tạo bằng lớp `Agent` từ `taskflowai` — bạn định nghĩa vai trò, mục tiêu, thuộc tính (attributes), model LLM và các tool mà agent có thể sử dụng.

Ví dụ khái quát (không phải toàn bộ thư viện):

```python
# Tạo agent bằng taskflowai
agent = Agent(
	role="Travel Agent",
	goal="Assist travelers with their queries",
	attributes="friendly, hardworking",
	llm=LoadModel.load_openai_model(),
	tools=[SearchFlights.search_flights_tool(), GetWeatherData.fetch_weather_data()]
)
```

## 2. Vị trí các phần quan trọng trong repository

- `src/agentic/agents/`: chứa các agent chính (ví dụ: `travel_agent.py`, `reporter_agent.py`, `web_research_agent.py`).
- `src/agentic/tools/`: chứa các tool độc lập (tương tác web, tìm flight, lấy weather, v.v.).
- `src/agentic/utils/`: các tiện ích chung, ví dụ `main_utils.py` có hàm load model.
- `deployment/`, `Dockerfile`, `demo.py`, `notebooks/`: tài liệu và cách chạy hệ thống.

## 3. Ví dụ thực tế từ code (step-by-step giải thích)

1) TravelAgent — nơi khởi tạo agent chuyên tìm thông tin du lịch

```python
class TravelAgent:
	@classmethod
	def initialize_travel_agent(cls):
		travel_agent = Agent(
			role="Travel Agent",
			goal="Assist travelers with their queries",
			attributes="friendly, hardworking, and detailed in reporting back to users",
			llm=LoadModel.load_openai_model(),
			tools=[SearchFlights.search_flights_tool(), GetWeatherData.fetch_weather_data()]
		)
		return travel_agent
```

Giải thích ngắn:
- `LoadModel.load_openai_model()` (xem `src/agentic/utils/main_utils.py`) trả model LLM đã cấu hình.
- `SearchFlights.search_flights_tool()` (xem `src/agentic/tools/search_flights.py`) là wrapper đưa API Amadeus vào danh sách tools của agent.

2) ReporterAgent — viết báo cáo từ kết quả

```python
class TravelReportAgent:
	@classmethod
	def initialize_travel_report_agent(cls):
		travel_report_agent = Agent(
			role="Travel Report Agent",
			goal="Write comprehensive travel reports with visual elements",
			attributes="friendly, hardworking, visual-oriented, and detailed in reporting",
			llm=LoadModel.load_openai_model()
		)
		return travel_report_agent
```

3) WebResearchAgent — tích hợp tìm kiếm web và images

```python
class WebResearchAgent:
	@classmethod
	def initialize_web_research_agent(cls):
		web_research_agent = Agent(
			role="Web Research Agent",
			goal="Research destinations and find relevant images",
			llm=LoadModel.load_openai_model(),
			tools=[SerperSearch.search_web(), WikiArticles.fetch_articles(), WikiImages.search_images()]
		)
		return web_research_agent
```

## 4. Pattern: Tools (tách riêng để dễ test & reuse)

- Mỗi tool là một wrapper nhỏ trả về một hàm hoặc callable mà `Agent` có thể gọi. Ví dụ `SearchFlights.search_flights_tool()` lấy `AmadeusTools.search_flights` và đóng gói nó.

```python
class SearchFlights:
	@classmethod
	def search_flights_tool(cls):
		search_flights = AmadeusTools.search_flights
		return search_flights
```

## 5. Quy trình để một fresher bắt đầu đọc/mở rộng

1. Mở `src/agentic/agents/travel_agent.py`: xem cách agent được khởi tạo.  
2. Mở `src/agentic/tools/`: học cách viết wrapper cho tool bên ngoài.  
3. Mở `src/agentic/utils/main_utils.py`: hiểu cách load model và validate env vars.  
4. Thử chạy `demo.py` hoặc notebook để quan sát luồng (input → agent → tool → output).

## 6. Gợi ý nhỏ khi mở rộng

- Thêm tool mới: tạo file trong `src/agentic/tools/`, viết wrapper như ví dụ, import vào agent và thêm vào danh sách `tools` khi khởi tạo agent.  
- Thêm agent mới: tạo file trong `src/agentic/agents/`, định nghĩa `initialize_*_agent()` giống mẫu.

---

Nếu muốn, tôi sẽ tiếp tục viết bài tiếp theo: **Kiến trúc agent** (luồng dữ liệu và ví dụ gọi tool), hoặc mở rộng bài này với hướng dẫn chạy `demo.py`.
