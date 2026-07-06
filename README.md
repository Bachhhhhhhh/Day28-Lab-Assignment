# Lab 28: AI Platform Integration Sprint (End-to-End)

## Overview
This repository contains a full end-to-end AI platform integrating Kafka, Prefect, Delta Lake, Feast, Qdrant, Prometheus, Grafana, API Gateway, and vLLM. It follows a hybrid architecture using local Docker Compose and Kaggle GPUs (tunneling via ngrok/cloudflared).

## Submission Status
✅ **Integration Completeness (40%)**: All 10 integration points successfully run. Data flows end-to-end from Kafka to Qdrant and LLM response.
✅ **Observability (25%)**: Metrics exported to Prometheus, Grafana dashboards active, and LangSmith traces configured (if valid API key is supplied).
✅ **Performance (20%)**: API Gateway is load-tested, and latency has been kept within SLO targets.
✅ **Architecture Quality (15%)**: API Gateway is wrapped with robust Pydantic validation and Try-Except error handling (Graceful degradation).

## Test Results
- **Smoke Tests**: 5/5 test classes passed (`pytest smoke-tests/ -v`).
- **Production Readiness**: Score 100% (10/10 checks passed).

## How to Run

1. Clone repo and create environment:
   ```bash
   python -m venv .venv
   .venv\Scripts\Activate.ps1
   pip install -r api-gateway/requirements.txt
   ```
2. Start Docker infrastructure:
   ```bash
   docker compose up -d
   ```
3. Update `.env` with your active `ngrok` (Embedding) and `cloudflared` (vLLM) URLs.
4. Restart API Gateway to load new variables:
   ```bash
   docker compose up -d api-gateway
   ```
5. Run tests:
   ```bash
   python -m pytest smoke-tests/ -v
   python scripts/production_readiness_check.py
   ```

## 5 Questions Answers

**Q1: Trade-offs trong thiết kế kiến trúc?**
- *Performance vs Reliability*: Sử dụng async HTTP cho tốc độ cao, nhưng phải thêm timeout handling/retry để không bị treo.
- *Cost vs Performance*: Tách GPU lên Kaggle (free) thay vì dùng máy vật lý giúp tiết kiệm cost nhưng bị độ trễ mạng (latency).

**Q2: Xử lý ngắt kết nối Local ↔ Kaggle thế nào?**
- Có cơ chế graceful degradation (try/catch) trong API Gateway. Nếu Qdrant hoặc vLLM lỗi/timeout (do ngrok bị sập), Gateway sẽ catch exception và trả về lỗi 503 thay vì crash toàn bộ service.

**Q3: Tại sao dùng Event-driven architecture với Kafka ở đầu pipeline?**
- Để *decoupling*: Producer đẩy data không cần quan tâm khi nào Prefect xử lý. Tăng cường *reliability*: Data lưu vào disk Kafka, không mất nếu pipeline down.

**Q4: Nêu các công cụ Observability đã cài đặt?**
- **Metrics**: Dùng `prometheus-fastapi-instrumentator` phơi `/metrics` cho Prometheus kéo (scrape).
- **Dashboards**: Grafana nhận data từ Prometheus.
- **Traces**: MLflow (Kaggle) tracking các thông số inference; LangSmith (Local) trace LangChain components.

**Q5: Graceful degradation trong hệ thống là gì?**
- Là việc khi một module lỗi, toàn bộ app không bị sập. Ví dụ: Nếu Qdrant bị sập, API Gateway sẽ bỏ qua Vector Search và truyền prompt không có context vào thẳng LLM (hoặc báo lỗi 503 gọn gàng). Code ở `api-gateway/main.py` đã áp dụng điều này.
