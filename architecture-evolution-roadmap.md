# GenAI Chatbot Platform - 아키텍처 발전 로드맵

## 🎯 현재 상태 → 미래 비전

### **Phase 0: 현재 (Monolithic)**
```
Single FastAPI App
├── Frontend (Streamlit)
└── Backend (FastAPI) - 모든 기능 포함
```

### **Phase 1: 모듈화 (3개월)**
```
Modular Monolith
├── Frontend (Streamlit)
└── Backend (FastAPI)
    ├── routers/     # 엔드포인트 분리
    ├── services/    # 비즈니스 로직
    ├── middleware/  # 공통 미들웨어
    └── infrastructure/ # 인프라 코드
```

### **Phase 2: 도메인 분리 (6개월)**
```
Domain-Driven Monolith
├── Frontend (Streamlit)
└── Backend (FastAPI)
    ├── auth/        # 인증 도메인
    ├── chat/        # 채팅 도메인
    ├── knowledge/   # 지식 검색 도메인
    ├── analytics/   # 분석 도메인 (신규)
    └── shared/      # 공통 도메인
```

### **Phase 3: 마이크로서비스 (12개월)**
```
Microservices Architecture
├── Frontend (Next.js/React)
├── API Gateway
├── Auth Service
├── Chat Service
├── Knowledge Service
├── Analytics Service
└── Notification Service (신규)
```

## 🏗️ Phase별 상세 계획

### **Phase 1: 모듈화 리팩토링 (즉시 시작)**

#### **1.1 디렉토리 구조 개선**
```python
src/
├── app.py                    # FastAPI 앱 설정
├── routers/
│   ├── __init__.py
│   ├── chat_router.py       # 채팅 엔드포인트
│   ├── auth_router.py       # 인증 엔드포인트
│   ├── feedback_router.py   # 피드백 엔드포인트
│   └── health_router.py     # 헬스체크
├── middleware/
│   ├── __init__.py
│   ├── guardrail.py         # 콘텐츠 필터링
│   ├── rate_limiting.py     # 속도 제한 (신규)
│   └── logging.py           # 로깅 미들웨어 (신규)
├── infrastructure/
│   ├── __init__.py
│   ├── bedrock_client.py    # AWS Bedrock 클라이언트
│   ├── dynamodb_client.py   # DynamoDB 클라이언트 (신규)
│   └── secrets_client.py    # Secrets Manager 클라이언트 (신규)
├── services/               # 기존 유지
├── adapters/              # 기존 유지
├── tools/                 # 기존 유지
└── utils/                 # 기존 유지
```

#### **1.2 의존성 주입 패턴 도입**
```python
# infrastructure/container.py
from dependency_injector import containers, providers
from src.services.chat_service import ChatService
from src.infrastructure.bedrock_client import BedrockClientManager

class Container(containers.DeclarativeContainer):
    # 설정
    config = providers.Configuration()
    
    # 인프라
    bedrock_client = providers.Singleton(BedrockClientManager)
    
    # 서비스
    chat_service = providers.Factory(
        ChatService,
        bedrock_client=bedrock_client
    )

# routers/chat_router.py
from dependency_injector.wiring import inject, Provide
from src.infrastructure.container import Container

@router.post("/chat")
@inject
async def chat(
    chat_request: ChatRequest,
    chat_service: ChatService = Provide[Container.chat_service]
):
    return await chat_service.process_chat(chat_request)
```

### **Phase 2: 도메인 중심 설계 (3-6개월)**

#### **2.1 도메인별 재구성**
```python
src/
├── domains/
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── models.py        # 인증 도메인 모델
│   │   ├── services.py      # 인증 비즈니스 로직
│   │   ├── repositories.py  # 인증 데이터 접근
│   │   └── routers.py       # 인증 API
│   ├── chat/
│   │   ├── __init__.py
│   │   ├── models.py        # 채팅 도메인 모델
│   │   ├── services.py      # 채팅 비즈니스 로직
│   │   ├── repositories.py  # 채팅 데이터 접근
│   │   └── routers.py       # 채팅 API
│   ├── knowledge/
│   │   ├── __init__.py
│   │   ├── models.py        # 지식 도메인 모델
│   │   ├── services.py      # 검색 비즈니스 로직
│   │   ├── repositories.py  # 지식베이스 접근
│   │   └── routers.py       # 검색 API
│   └── analytics/           # 신규 도메인
│       ├── __init__.py
│       ├── models.py        # 분석 모델
│       ├── services.py      # 분석 로직
│       └── routers.py       # 분석 API
├── shared/
│   ├── events/              # 도메인 이벤트
│   ├── exceptions/          # 공통 예외
│   └── middleware/          # 공통 미들웨어
└── infrastructure/          # 기존 유지
```

#### **2.2 이벤트 기반 통신 도입**
```python
# shared/events/base.py
from abc import ABC, abstractmethod
from datetime import datetime
from typing import Dict, Any

class DomainEvent(ABC):
    def __init__(self):
        self.occurred_at = datetime.utcnow()
        self.event_id = str(uuid.uuid4())

class ChatMessageReceived(DomainEvent):
    def __init__(self, session_id: str, message: str, user_id: str):
        super().__init__()
        self.session_id = session_id
        self.message = message
        self.user_id = user_id

# shared/events/bus.py
class EventBus:
    def __init__(self):
        self._handlers = {}
    
    def subscribe(self, event_type: type, handler: callable):
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)
    
    async def publish(self, event: DomainEvent):
        handlers = self._handlers.get(type(event), [])
        for handler in handlers:
            await handler(event)

# domains/analytics/services.py
class AnalyticsService:
    async def handle_chat_message(self, event: ChatMessageReceived):
        # 채팅 메시지 분석 로직
        await self.track_user_interaction(event.user_id, event.message)
```

### **Phase 3: 마이크로서비스 전환 (6-12개월)**

#### **3.1 서비스 분리 전략**
```yaml
# docker-compose.yml
version: '3.8'
services:
  # API Gateway
  api-gateway:
    image: nginx:alpine
    ports:
      - "80:80"
    depends_on:
      - auth-service
      - chat-service
      - knowledge-service

  # 인증 서비스
  auth-service:
    build: ./services/auth-service
    environment:
      - DATABASE_URL=postgresql://auth_db:5432/auth
      - JWT_SECRET_ARN=arn:aws:secretsmanager:...
    depends_on:
      - auth-db

  # 채팅 서비스
  chat-service:
    build: ./services/chat-service
    environment:
      - BEDROCK_REGION=ap-northeast-2
      - KNOWLEDGE_SERVICE_URL=http://knowledge-service:8000
    depends_on:
      - chat-db

  # 지식 검색 서비스
  knowledge-service:
    build: ./services/knowledge-service
    environment:
      - KNOWLEDGE_BASE_ID=AVMYFLHAD9
      - OPENSEARCH_ENDPOINT=https://...

  # 데이터베이스들
  auth-db:
    image: postgres:15
  chat-db:
    image: postgres:15
```

#### **3.2 서비스 간 통신**
```python
# services/chat-service/src/clients/knowledge_client.py
import httpx
from typing import List, Dict

class KnowledgeServiceClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.client = httpx.AsyncClient()
    
    async def search_documents(self, query: str, group: str) -> List[Dict]:
        response = await self.client.post(
            f"{self.base_url}/search",
            json={"query": query, "group": group}
        )
        return response.json()

# services/chat-service/src/services/chat_service.py
class ChatService:
    def __init__(self, knowledge_client: KnowledgeServiceClient):
        self.knowledge_client = knowledge_client
    
    async def generate_response(self, message: str, group: str):
        # 지식 검색 서비스 호출
        documents = await self.knowledge_client.search_documents(message, group)
        
        # LLM 호출 및 응답 생성
        return await self.llm.generate(message, documents)
```

## 🔄 서비스 간 통신 패턴

### **1. 동기 통신 (HTTP/gRPC)**
```python
# 실시간 응답이 필요한 경우
chat_service → knowledge_service (문서 검색)
chat_service → auth_service (토큰 검증)
```

### **2. 비동기 통신 (Message Queue)**
```python
# 이벤트 기반 처리
chat_service → analytics_service (사용자 행동 분석)
chat_service → notification_service (알림 발송)

# AWS SQS/SNS 활용
import boto3

class EventPublisher:
    def __init__(self):
        self.sns = boto3.client('sns')
    
    async def publish_chat_event(self, event_data: dict):
        await self.sns.publish(
            TopicArn='arn:aws:sns:ap-northeast-2:...:chat-events',
            Message=json.dumps(event_data)
        )
```

## 📊 관리 효율성 향상 방안

### **1. 통합 모니터링**
```yaml
# observability/docker-compose.yml
services:
  # 메트릭 수집
  prometheus:
    image: prom/prometheus
    
  # 로그 수집
  loki:
    image: grafana/loki
    
  # 시각화
  grafana:
    image: grafana/grafana
    
  # 분산 추적
  jaeger:
    image: jaegertracing/all-in-one
```

### **2. 자동화된 배포**
```yaml
# .github/workflows/deploy.yml
name: Deploy Services
on:
  push:
    branches: [main]

jobs:
  deploy-auth-service:
    if: contains(github.event.head_commit.modified, 'services/auth-service/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to ECS
        run: |
          aws ecs update-service --service auth-service --force-new-deployment
```

### **3. 설정 관리**
```python
# shared/config/base.py
from pydantic import BaseSettings

class BaseConfig(BaseSettings):
    service_name: str
    log_level: str = "INFO"
    aws_region: str = "ap-northeast-2"
    
    # 서비스 디스커버리
    auth_service_url: str
    knowledge_service_url: str
    
    class Config:
        env_file = ".env"

# services/chat-service/config.py
class ChatServiceConfig(BaseConfig):
    service_name: str = "chat-service"
    bedrock_model_id: str
    max_tokens: int = 2048
```

## 🎯 마이그레이션 우선순위

### **즉시 (1-2주)**
1. ✅ 라우터 분리 (`routers/`)
2. ✅ 미들웨어 분리 (`middleware/`)
3. ✅ 인프라 코드 분리 (`infrastructure/`)

### **단기 (1-3개월)**
1. 🔄 의존성 주입 패턴 도입
2. 🔄 이벤트 기반 아키텍처 준비
3. 🔄 모니터링 및 로깅 강화

### **중기 (3-6개월)**
1. 📋 도메인별 재구성
2. 📋 분석 도메인 추가
3. 📋 성능 최적화

### **장기 (6-12개월)**
1. 🎯 마이크로서비스 분리
2. 🎯 컨테이너 오케스트레이션
3. 🎯 서비스 메시 도입

## 🚀 성공 지표

- **코드 품질**: 순환 복잡도 < 10, 테스트 커버리지 > 80%
- **성능**: 응답 시간 < 2초, 처리량 > 1000 RPS
- **안정성**: 가용성 > 99.9%, 에러율 < 0.1%
- **개발 효율성**: 배포 시간 < 10분, 롤백 시간 < 5분