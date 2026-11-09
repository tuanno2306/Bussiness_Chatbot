---
name: fastapi-architecture
description: >-
  Enterprise-grade FastAPI backend patterns, project structures, async streaming (SSE),
  Redis session conversation memory, rate limiting, and Docker deployment.
  Use when writing FastAPI endpoints, setting up chat streaming, structuring microservices, or deploying backend containers.
---

# FastAPI Enterprise Architecture Skill

This skill provides production-grade architectural guidelines for building high-performance, asynchronous Python backend services for AI chatbots.

---

## 1. Project Directory Structure

```text
backend/
├── app/
│   ├── api/                  # API routers (endpoints)
│   │   ├── v1/
│   │   │   ├── chat.py       # Chat completion & streaming endpoints
│   │   │   ├── documents.py  # Knowledge ingestion (upload, index, delete)
│   │   │   ├── tools.py      # Tool integrations & webhooks
│   │   │   └── health.py     # Liveness & readiness probes
│   │   └── router.py         # Consolidated API router
│   ├── core/                 # App-wide configurations
│   │   ├── config.py         # Pydantic Settings (env variables)
│   │   ├── security.py       # API Key auth & JWT validation
│   │   └── redis.py          # Redis client for sessions & rate limiting
│   ├── models/               # Data structures
│   │   ├── domain.py         # Pydantic request/response schemas
│   │   └── db.py             # SQLAlchemy/SQLModel database entities
│   ├── services/             # Business & AI logic
│   │   ├── rag_service.py    # Vector search & re-ranking logic
│   │   ├── llm_service.py    # Multi-provider LLM caller (Gemini/OpenAI)
│   │   └── memory_service.py # Redis multi-turn session history
│   └── main.py               # Application factory & lifespan setup
├── tests/                    # Pytest test suite
├── Dockerfile                # Multi-stage Docker build
├── docker-compose.yml        # Local stack (FastAPI + Qdrant + Redis + Postgres)
└── requirements.txt          # Python dependencies
```

---

## 2. Server-Sent Events (SSE) Streaming Response

Để chatbot hiển thị từng từ (typing effect) mượt mà tới khách hàng, luôn sử dụng SSE streaming thay vì chờ sinh toàn bộ câu trả lời:

```python
# app/api/v1/chat.py
from fastapi import APIRouter, Depends
from fastapi.responses import StreamingResponse
from app.models.domain import ChatRequest
from app.services.llm_service import stream_chat_response

router = APIRouter(prefix="/chat", tags=["Chat"])

@router.post("/stream")
async def chat_stream_endpoint(request: ChatRequest):
    """Streaming endpoint for Web Widget using SSE"""
    
    async def event_generator():
        async for chunk in stream_chat_response(
            session_id=request.session_id,
            message=request.message
        ):
            # Format chuẩn SSE: "data: <content>\n\n"
            yield f"data: {chunk}\n\n"
            
        yield "data: [DONE]\n\n"

    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

---

## 3. Session & Multi-turn Memory với Redis

```python
# app/services/memory_service.py
import json
import redis.asyncio as aioredis
from app.core.config import settings

redis_client = aioredis.from_url(settings.REDIS_URL, decode_responses=True)

async def get_chat_history(session_id: str, limit: int = 6) -> list[dict]:
    """Lấy N lượt hội thoại gần nhất để đưa vào ngữ cảnh LLM"""
    key = f"chat_session:{session_id}"
    raw_history = await redis_client.lrange(key, -limit, -1)
    return [json.loads(item) for item in raw_history]

async def add_chat_message(session_id: str, role: str, content: str, ttl_seconds: int = 86400):
    """Lưu tin nhắn mới và gia hạn thời gian sống của session (24h)"""
    key = f"chat_session:{session_id}"
    message = json.dumps({"role": role, "content": content})
    await redis_client.rpush(key, message)
    await redis_client.expire(key, ttl_seconds)
```

---

## 4. Production Readiness Checklist

1. **Lifespan Manager:** Sử dụng `@asynccontextmanager` trong `main.py` để khởi tạo kết nối Database, Qdrant và Redis trước khi nhận request.
2. **CORS:** Cấu hình nguồn gốc hợp lệ (`allow_origins`) cho Web Chat Widget từ domain của công ty.
3. **Rate Limiting:** Sử dụng `slowapi` hoặc Redis Token Bucket để giới hạn tối đa 20 tin nhắn/phút cho mỗi IP/Session tránh lạm dụng.
4. **Health Check:** Luôn có endpoint `GET /health` để Kubernetes hoặc Docker Compose kiểm tra trạng thái hoạt động của dịch vụ.
