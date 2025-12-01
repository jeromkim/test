# Backend 파일 명명 규칙

## 🎯 권장 구조

### **명확한 파일명으로 분리**
```
backend-fastapi/
├── src/
│   ├── server.py           # FastAPI 앱 설정 (main.py 대신)
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── chat_router.py
│   │   ├── auth_router.py
│   │   └── health_router.py
│   ├── middleware/
│   │   ├── __init__.py
│   │   └── guardrail.py
│   └── infrastructure/
│       ├── __init__.py
│       └── bedrock_client.py
├── main.py                 # 엔트리포인트 (server.py import)
└── pyproject.toml

frontend-streamlit/
├── app.py                  # Streamlit 메인 앱
├── pages/
└── utils/
```

## 📁 파일별 역할

### **main.py (엔트리포인트)**
```python
# 단순한 엔트리포인트 (5줄)
from src.server import app

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### **src/server.py (FastAPI 앱 설정)**
```python
# FastAPI 앱 생성 및 설정 (30줄)
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from src.routers import chat_router, auth_router, health_router
from src.infrastructure.bedrock_client import lifespan

def create_app() -> FastAPI:
    app = FastAPI(title="GenAI Chatbot API", lifespan=lifespan)
    
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

## 🔄 다른 명명 옵션들

### **Option A: 프레임워크 명시**
```
src/fastapi_app.py    # 명확하지만 길음
src/api_server.py     # API 서버임을 명시
src/web_server.py     # 웹 서버임을 명시
```

### **Option B: 도메인 명시**
```
src/chatbot_server.py # 도메인 특화
src/genai_app.py      # 프로젝트 특화
```

### **Option C: 간단한 명명**
```
src/server.py         # 가장 간단하고 명확 ✅
src/api.py           # API 앱임을 명시
src/backend.py       # 백엔드임을 명시
```

## 🎯 권장 이유: server.py

### **장점 ✅**
1. **명확성**: 서버 설정 파일임을 명확히 표현
2. **간결성**: 짧고 기억하기 쉬움
3. **일반성**: 프레임워크에 종속되지 않음
4. **구분성**: Frontend app.py와 명확히 구분

### **사용 예시**
```python
# 개발 시
uvicorn src.server:app --reload

# 배포 시  
uvicorn src.server:app --host 0.0.0.0 --port 8000

# 테스트 시
from src.server import app
```

## 📋 전체 프로젝트 구조

```
genai-chatbot-platform/
├── app/
│   ├── backend-fastapi/
│   │   ├── src/
│   │   │   ├── server.py        # ✅ FastAPI 앱 설정
│   │   │   ├── routers/
│   │   │   ├── middleware/
│   │   │   └── infrastructure/
│   │   ├── main.py              # 엔트리포인트
│   │   └── pyproject.toml
│   └── frontend-streamlit/
│       ├── app.py               # ✅ Streamlit 메인 앱
│       ├── pages/
│       └── utils/
├── docs/
└── architecture/
```

## 🚀 마이그레이션 단계

### **Step 1: server.py 생성**
```python
# 현재 main.py의 FastAPI 앱 설정 부분을 src/server.py로 이동
```

### **Step 2: main.py 단순화**
```python
# main.py를 단순한 엔트리포인트로 변경
from src.server import app
```

### **Step 3: 라우터 분리**
```python
# 각 엔드포인트를 routers/ 디렉토리로 분리
```

## ✅ 최종 권장사항

**`src/server.py`**를 사용하는 것이 가장 적합합니다:
- Frontend `app.py`와 명확히 구분
- 서버 설정 파일임을 직관적으로 표현
- 간결하고 기억하기 쉬움
- 프레임워크에 종속되지 않음