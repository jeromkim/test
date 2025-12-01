# main.py 리팩토링 가이드

## 🎯 분리 목표

현재 main.py (200+ 줄) → 역할별 모듈 분리

## 📁 새로운 구조

```
src/
├── app.py                    # FastAPI 앱 설정 (30줄)
├── routers/
│   ├── __init__.py
│   ├── chat_router.py       # 채팅 관련 엔드포인트 (80줄)
│   ├── auth_router.py       # 인증 엔드포인트 (이미 존재)
│   └── health_router.py     # 헬스체크 엔드포인트 (10줄)
├── middleware/
│   ├── __init__.py
│   └── guardrail.py         # Guardrail 미들웨어 (50줄)
└── infrastructure/
    ├── __init__.py
    └── bedrock_client.py    # AWS 클라이언트 관리 (40줄)
```

## 🔧 분리 상세 계획

### 1. **app.py** (FastAPI 앱 설정만)
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from src.routers import chat_router, auth_router, health_router
from src.infrastructure.bedrock_client import lifespan

def create_app() -> FastAPI:
    app = FastAPI(title="Lotte Mart Chatbot API", lifespan=lifespan)
    
    # CORS 설정
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_methods=["*"],
        allow_headers=["*"],
    )
    
    # 라우터 등록
    app.include_router(auth_router.router, prefix="/api")
    app.include_router(chat_router.router, prefix="/api")
    app.include_router(health_router.router)
    
    return app

app = create_app()
```

### 2. **routers/chat_router.py** (채팅 엔드포인트)
```python
from fastapi import APIRouter, Request
from src.utils.models import ChatRequest
from src.adapters.chat_controller import handle_chat_request
from src.services.chat_service import ChatService
from src.middleware.guardrail import check_input_guardrail

router = APIRouter()

@router.post("/chat")
async def chat(chat_request: ChatRequest, request: Request):
    user_ip = get_client_ip(request)
    
    # Guardrail 체크
    guardrail_response = await check_input_guardrail(chat_request.user_message_content)
    if guardrail_response["action"] == "GUARDRAIL_INTERVENED":
        return await create_guardrail_blocked_response(guardrail_response)
    
    # 채팅 서비스 생성
    chat_service = ChatService(
        model=chat_request.model_id or config.model_id,
        temperature=MODEL_TEMPERATURE,
        max_tokens=MODEL_MAX_TOKENS,
        group=chat_request.group or "common",
        user_prompt=chat_request.user_prompt
    )
    
    # 채팅 처리
    return await handle_chat_request(
        session_id=chat_request.session_id,
        user_message_content=chat_request.user_message_content,
        user_message_id=chat_request.user_message_id,
        ai_response_message_id=chat_request.ai_response_message_id,
        user_ip=user_ip,
        stream=chat_request.stream,
        chat_service=chat_service,
        group=chat_request.group or "common"
    )

@router.post("/chat/compliance")
async def chat_compliance(chat_request: ChatRequest, request: Request):
    # 컴플라이언스 채팅 로직
    pass

@router.post("/feedback")
async def post_feedback(request: Request):
    # 피드백 로직
    pass
```

### 3. **middleware/guardrail.py** (Guardrail 미들웨어)
```python
import json
from starlette.responses import StreamingResponse
from src.infrastructure.bedrock_client import get_bedrock_client
from src.config import config

async def check_input_guardrail(user_message_content: str):
    """입력 메시지에 대한 Guardrail 체크"""
    bedrock_client = get_bedrock_client('ap-northeast-2')
    
    response = await bedrock_client.apply_guardrail(
        guardrailIdentifier=config.guardrail_id,
        guardrailVersion=config.guardrail_version,
        source='INPUT',
        content=[{
            'text': {'text': user_message_content}
        }]
    )
    return response

async def create_guardrail_blocked_response(guardrail_response):
    """Guardrail 차단 시 SSE 응답 생성"""
    async def sse_generator():
        yield f"data: {json.dumps({'role': 'assistant', 'content': guardrail_response['outputs'][0]['text']}, ensure_ascii=False)}\n\n"
    
    return StreamingResponse(
        sse_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no"
        }
    )
```

### 4. **infrastructure/bedrock_client.py** (AWS 클라이언트 관리)
```python
from contextlib import asynccontextmanager
import aioboto3
from src.config import config
from src.utils.logger import logger

# 전역 클라이언트 풀
bedrock_clients = {}

@asynccontextmanager
async def lifespan(app):
    """앱 시작 시 Bedrock 클라이언트 초기화, 종료 시 정리"""
    logger.info("Initializing Bedrock clients...")
    
    # 서울 리전
    seoul_session = aioboto3.Session(region_name=config.aws_region)
    bedrock_clients['ap-northeast-2'] = await seoul_session.client("bedrock-runtime").__aenter__()
    
    # 미국 서부 리전
    us_west_session = aioboto3.Session(region_name="us-west-2")
    bedrock_clients['us-west-2'] = await us_west_session.client("bedrock-runtime").__aenter__()
    
    # 미국 동부 리전
    us_east_session = aioboto3.Session(region_name="us-east-1")
    bedrock_clients['us-east-1'] = await us_east_session.client("bedrock-runtime").__aenter__()
    
    logger.info("All Bedrock clients initialized successfully")
    
    yield
    
    # 종료 시 정리
    logger.info("Closing Bedrock clients...")
    for region, client in bedrock_clients.items():
        try:
            await client.__aexit__(None, None, None)
            logger.info(f"Closed client for region: {region}")
        except Exception as e:
            logger.error(f"Error closing client for {region}: {e}")

def get_bedrock_client(region: str):
    """특정 리전의 Bedrock 클라이언트 반환"""
    return bedrock_clients.get(region)
```

### 5. **routers/health_router.py** (헬스체크)
```python
from fastapi import APIRouter

router = APIRouter()

@router.get("/health")
async def health_check():
    """서버 상태 확인"""
    return {"status": "healthy"}
```

## 🚀 마이그레이션 단계

### **Step 1: 새 파일들 생성**
```bash
mkdir -p src/routers src/middleware src/infrastructure
touch src/routers/__init__.py
touch src/middleware/__init__.py  
touch src/infrastructure/__init__.py
```

### **Step 2: 코드 이동**
1. `infrastructure/bedrock_client.py` 생성
2. `middleware/guardrail.py` 생성
3. `routers/chat_router.py` 생성
4. `routers/health_router.py` 생성
5. `app.py` 생성

### **Step 3: main.py 교체**
```python
# 새로운 main.py (5줄)
from src.app import app

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## ✅ 분리 후 장점

1. **가독성**: 각 파일이 30-80줄로 관리 용이
2. **테스트**: 각 모듈별 독립적 테스트 가능
3. **유지보수**: 변경 시 영향 범위 최소화
4. **재사용**: 미들웨어, 인프라 코드 재사용 가능
5. **확장성**: 새로운 라우터 추가 용이