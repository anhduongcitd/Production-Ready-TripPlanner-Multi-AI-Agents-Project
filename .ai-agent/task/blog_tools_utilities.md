# Bài 6 — Tools & Utilities (dành cho Fresher)

Mục tiêu: Giải thích pattern viết `tools` trong repo, cách tổ chức wrapper cho API ngoài, quy ước trả dữ liệu, kiểm thử và best practices để mở rộng an toàn.

---

## 1) Tại sao tách `tools` ra riêng?

- Độc lập hoá phần gọi API bên ngoài khỏi logic agent.
- Dễ test (mock) và tái sử dụng giữa nhiều agent.
- Giảm phạm vi cần mock khi viết unit tests cho agent.

## 2) Các file ví dụ trong repo

- `src/agentic/tools/search_flights.py` (Amadeus)
- `src/agentic/tools/get_weather_data.py` (weather API)
- `src/agentic/tools/serper_search.py` (Serper)
- `src/agentic/tools/search_articles.py` (Wikipedia)
- `src/agentic/tools/search_images.py` (Wikipedia)

## 3) Pattern chung (template)

- Mỗi tool là một lớp chứa phương thức `@classmethod` trả về một callable (hàm) hoặc trực tiếp trả API callable từ thư viện bên ngoài.
- Wrapper nên:
  - Log input/output ở mức debug
  - Bọc lỗi vào `CustomException`
  - Trả dữ liệu có cấu trúc rõ ràng (JSON/dict/list) hoặc trả markdown nếu tool tạo nội dung để hiển thị trực tiếp

Template:

```python
from src.agentic.logger import logging
from src.agentic.exception import CustomException
import sys

class MyTool:
    @classmethod
    def my_callable(cls):
        try:
            logging.info("Initializing MyTool")

            def call(params):
                # gọi API hoặc xử lý
                result = {"ok": True, "data": []}
                return result

            return call

        except Exception as e:
            logging.error("MyTool failed: %s", str(e))
            raise CustomException(sys, e)
```

## 4) Hợp đồng đầu ra (Output contract)

- Định dạng có cấu trúc (máy dễ parse):
  - `dict` với các key rõ ràng (ví dụ `{"flights": [{"carrier":..., "price":...}]}`)
- Nếu output dùng trực tiếp cho UI (báo cáo), có thể trả `markdown` string chứa `![Alt](https://...)` cho ảnh.
- Luôn đồng bộ với việc agent mong đợi: agent sẽ dùng LLM để tổng hợp nếu input có cấu trúc, hoặc render markdown trực tiếp nếu content là markdown.

## 5) Xử lý lỗi & logging

- Dùng `CustomException` (repo có sẵn) để bọc lỗi và giữ stack trace.
- Log input/output ở mức `debug`; mask secrets khi log.
- Trả lỗi thân thiện cho UI, ví dụ `{"error": "Service unavailable, try later"}`.

## 6) Kiểm thử tool (mocking)

- Unit tests: mock API client hoặc mock callable trả về mẫu dữ liệu.

Ví dụ pytest cho `search_flights`:

```python
from src.agentic.tools.search_flights import SearchFlights
from unittest.mock import patch

@patch('taskflowai.AmadeusTools.search_flights')
def test_search_flights(mock_amadeus):
    mock_amadeus.return_value = [{"flight": "ABC123", "price": 120}]
    fn = SearchFlights.search_flights_tool()
    res = fn({"from": "HAN", "to": "BKK"})
    assert isinstance(res, list)
    assert res[0]["price"] == 120
```

## 7) Quản lý secrets & env vars

- Các tools cần keys: `AMADEUS_API_KEY`, `AMADEUS_API_SECRET`, `WEATHER_API_KEY`, `SERPER_API_KEY`, `OPENAI_API_KEY`, v.v.
- Lưu trữ trong `.env` khi dev; ở production dùng secret manager (Vault, AWS Secrets Manager).
- Không log toàn bộ secret — mask trước khi ghi.

## 8) Hiệu suất & caching

- Thêm caching (in-memory LRU hoặc Redis) cho các API tốn phí.
- Giới hạn TTL cho cache và expose cách xoá cache khi cần.

## 9) Các lưu ý khi viết tool mới

1. Viết wrapper theo template trên.
2. Đảm bảo output contract rõ ràng: schema/dict hoặc markdown.
3. Viết unit test mock.
4. Thêm logging debug và mask secrets.
5. Tích hợp vào agent bằng cách import và thêm `MyTool.my_callable()` vào `tools` list.

## 10) Ví dụ: thêm tool đơn giản

```python
# src/agentic/tools/my_tool.py
from src.agentic.exception import CustomException
from src.agentic.logger import logging
import sys

class MyTool:
    @classmethod
    def my_callable(cls):
        try:
            def call(params):
                # giả lập xử lý
                return {"message": f"Received {params}"}
            return call
        except Exception as e:
            raise CustomException(sys, e)
```

Và import vào agent:

```python
from src.agentic.tools.my_tool import MyTool
# ... tools=[..., MyTool.my_callable()]
```

---

Tôi sẵn sàng tiếp tục: viết bài về `RAG Pipeline` kế tiếp, hoặc thêm ví dụ unit test chi tiết cho một tool bạn chọn.