# Bài 9 — Triển khai Docker (dành cho Fresher)

Mục tiêu: Hướng dẫn cách đóng gói và chạy ứng dụng TripPlanner (Streamlit + agents) bằng Docker, bao gồm các cách build, chạy, và lưu ý bảo mật biến môi trường.

---

## 1) Tổng quan về Dockerfile trong repo

Repo có sẵn `Dockerfile` ở project root. Tóm tắt:

- Base image: `python:3.10-slim`
- Cài dependencies từ `requirements.txt`
- Định nghĩa `ARG` cho các API key (OPENAI, WEATHER, SERPER, AMADEUS)
- Thiết lập `ENV` từ các `ARG`
- Expose port `8501` và chạy Streamlit entry: `deployment/app.py`

Nội dung chính file (`Dockerfile`):

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
ARG OPENAI_API_KEY
ARG WEATHER_API_KEY
ARG SERPER_API_KEY
ARG AMADEUS_API_KEY
ARG AMADEUS_API_SECRET
ENV OPENAI_API_KEY=${OPENAI_API_KEY}
ENV WEATHER_API_KEY=${WEATHER_API_KEY}
ENV SERPER_API_KEY=${SERPER_API_KEY}
ENV AMADEUS_API_KEY=${AMADEUS_API_KEY}
ENV AMADEUS_API_SECRET=${AMADEUS_API_SECRET}
EXPOSE 8501
CMD ["streamlit", "run", "deployment/app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

> Ghi chú: Dockerfile hiện đặt ENV từ ARG (baked at build-time). Thông thường **không nên** ghi thẳng API keys vào image (bảo mật). Thay vào đó, truyền biến môi trường khi `docker run` hoặc dùng secret manager.

## 2) Build & Run (2 cách)

A) (Không khuyến nghị) Build và bake keys vào image (bất lợi: key bị lưu trong image/ lịch sử build):

```bash
docker build -t tripplanner:latest \
  --build-arg OPENAI_API_KEY="${OPENAI_API_KEY}" \
  --build-arg WEATHER_API_KEY="${WEATHER_API_KEY}" \
  --build-arg SERPER_API_KEY="${SERPER_API_KEY}" \
  --build-arg AMADEUS_API_KEY="${AMADEUS_API_KEY}" \
  --build-arg AMADEUS_API_SECRET="${AMADEUS_API_SECRET}" \
  .
```

B) (Khuyến nghị) Build image không chứa key, truyền biến khi chạy container:

```bash
docker build -t tripplanner:latest .

docker run -p 8501:8501 \
  -e OPENAI_API_KEY="${OPENAI_API_KEY}" \
  -e WEATHER_API_KEY="${WEATHER_API_KEY}" \
  -e SERPER_API_KEY="${SERPER_API_KEY}" \
  -e AMADEUS_API_KEY="${AMADEUS_API_KEY}" \
  -e AMADEUS_API_SECRET="${AMADEUS_API_SECRET}" \
  tripplanner:latest
```

- `-p 8501:8501` ánh xạ port của container sang host để truy cập Streamlit.
- Bạn có thể dùng file `.env` và `docker-compose` để tổ chức nhiều dịch vụ.

## 3) Docker Compose (ví dụ)

Tạo `docker-compose.yml` (ví dụ) để chạy dễ dàng:

```yaml
version: '3.8'
services:
  tripplanner:
    build: .
    image: tripplanner:latest
    ports:
      - "8501:8501"
    environment:
      OPENAI_API_KEY: "${OPENAI_API_KEY}"
      WEATHER_API_KEY: "${WEATHER_API_KEY}"
      SERPER_API_KEY: "${SERPER_API_KEY}"
      AMADEUS_API_KEY: "${AMADEUS_API_KEY}"
      AMADEUS_API_SECRET: "${AMADEUS_API_SECRET}"
    restart: unless-stopped
```

Sử dụng `.env` bên cạnh `docker-compose.yml` để lưu biến môi trường (không commit `.env` vào git):

```
OPENAI_API_KEY=sk-...
WEATHER_API_KEY=...
SERPER_API_KEY=...
AMADEUS_API_KEY=...
AMADEUS_API_SECRET=...
```

Chạy:

```bash
docker compose up --build
```

## 4) Đẩy image lên registry (Docker Hub / GHCR)

```bash
docker tag tripplanner:latest youruser/tripplanner:1.0
docker push youruser/tripplanner:1.0
```

Trên server production, kéo image và chạy container, truyền biến môi trường từ secret manager hoặc CI/CD.

## 5) Best practices & lưu ý

- KHÔNG commit `.env` hoặc bake API keys vào image.
- Dùng secret manager (AWS Secrets Manager, GitHub Actions secrets) để inject env vars tại runtime hoặc CI.
- Giới hạn quyền cho API key (scope, rate limit) nếu có thể.
- Giới hạn resources (CPU/memory) cho container nếu chạy trên server với nhiều dịch vụ.
- Sử dụng healthchecks và restart policies trong `docker-compose`.

---

Bạn muốn tôi thêm `docker-compose.yml` mẫu vào repo hoặc tạo `README` ngắn hướng dẫn deploy không?