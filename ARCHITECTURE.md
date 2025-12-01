# GenAI Chatbot Application Architecture

## 📁 디렉토리 구조
```
src/
├── adapters/           # 외부 인터페이스 어댑터
│   └── chat_controller.py
├── infrastructure/     # 외부 시스템 연동
│   ├── __init__.py
│   └── bedrock_client.py
├── middleware/         # 횡단 관심사
│   ├── __init__.py
│   └── guardrail.py
├── routers/           # API 엔드포인트
│   ├── __init__.py
│   ├── auth_router.py
│   ├── chat_router.py
│   ├── feedback_router.py
│   └── health_router.py
├── services/          # 비즈니스 로직
│   ├── chat_service.py
│   ├── history_service.py
│   └── streaming_service.py
├── tools/             # LangChain 도구
│   └── retrieve_knowledge_search.py
├── utils/             # 공통 유틸리티
│   ├── logger.py
│   ├── models.py
│   └── performance_logger.py
├── dependencies.py    # 의존성 주입
├── exceptions.py      # 예외 처리
├── config.py         # 환경 설정
├── constant.py       # 상수 정의
└── server.py         # 앱 진입점
```

## 🏗️ 아키텍처 레이어

### **1. Presentation Layer (프레젠테이션)**
- **routers/**: FastAPI 라우터들
  - `auth_router.py` - JWT 인증
  - `chat_router.py` - 채팅 API
  - `feedback_router.py` - 피드백 수집
  - `health_router.py` - 헬스체크
- **server.py**: FastAPI 앱 설정 및 라우터 등록

### **2. Application Layer (애플리케이션)**
- **adapters/chat_controller.py**: 채팅 요청 처리 오케스트레이션
- **dependencies.py**: 의존성 주입 팩토리
- **exceptions.py**: 표준화된 예외 처리

### **3. Domain Layer (도메인)**
- **services/**: 핵심 비즈니스 로직
  - `chat_service.py` - LLM 채팅 처리
  - `history_service.py` - 대화 히스토리 관리
  - `streaming_service.py` - SSE 스트리밍 처리

### **4. Infrastructure Layer (인프라)**
- **infrastructure/bedrock_client.py**: AWS Bedrock 클라이언트 관리
- **middleware/guardrail.py**: 콘텐츠 필터링
- **tools/**: LangChain 도구 통합

### **5. Cross-Cutting Concerns (횡단 관심사)**
- **utils/**: 로깅, 모델, 성능 측정
- **config.py**: 환경 설정 관리
- **constant.py**: 상수 정의

## 🔄 데이터 플로우

```
Client Request → Router → Controller → Service → Infrastructure
                   ↓         ↓          ↓           ↓
               Validation  Business   Domain    External APIs
                          Logic      Logic     (Bedrock, DynamoDB)
```

## 🎯 핵심 패턴

### **의존성 주입**
- 팩토리 패턴으로 서비스 생성 표준화
- `@lru_cache()`로 싱글톤 관리

### **레이어 분리**
- ChatService: 순수 데이터 생성
- StreamingService: SSE 형식 변환
- Controller: 오케스트레이션

### **에러 처리**
- 구조화된 예외 (`error_type`, `message`)
- 세분화된 에러 타입별 처리

## 📊 Architecture Diagram

```mermaid
graph TB
    %% Client Layer
    Client[🌐 Client<br/>Streamlit Frontend]
    
    %% API Gateway
    subgraph "🚪 API Gateway"
        FastAPI[FastAPI Server<br/>server.py]
    end
    
    %% Presentation Layer
    subgraph "📡 Presentation Layer"
        AuthRouter[🔐 Auth Router<br/>JWT Verification]
        ChatRouter[💬 Chat Router<br/>Chat Endpoints]
        FeedbackRouter[📝 Feedback Router<br/>User Feedback]
        HealthRouter[❤️ Health Router<br/>Health Check]
    end
    
    %% Application Layer
    subgraph "🎯 Application Layer"
        ChatController[🎮 Chat Controller<br/>Request Orchestration]
        Dependencies[⚙️ Dependencies<br/>DI Container]
        Exceptions[⚠️ Exceptions<br/>Error Handling]
    end
    
    %% Domain Layer
    subgraph "🏢 Domain Layer"
        ChatService[🤖 Chat Service<br/>LLM Processing]
        HistoryService[📚 History Service<br/>Conversation Management]
        StreamingService[📡 Streaming Service<br/>SSE Processing]
    end
    
    %% Infrastructure Layer
    subgraph "🏗️ Infrastructure Layer"
        BedrockClient[☁️ Bedrock Client<br/>AWS Integration]
        Guardrail[🛡️ Guardrail<br/>Content Filtering]
        Tools[🔧 LangChain Tools<br/>Knowledge Search]
    end
    
    %% External Services
    subgraph "🌍 External Services"
        Bedrock[AWS Bedrock<br/>Claude 3.5]
        DynamoDB[DynamoDB<br/>Chat History]
        OpenSearch[OpenSearch<br/>Knowledge Base]
        SecretsManager[Secrets Manager<br/>JWT Secret]
    end
    
    %% Cross-Cutting
    subgraph "🔄 Cross-Cutting"
        Config[⚙️ Config<br/>Environment Settings]
        Logger[📝 Logger<br/>Structured Logging]
        Models[📋 Models<br/>Data Schemas]
    end
    
    %% Connections
    Client --> FastAPI
    FastAPI --> AuthRouter
    FastAPI --> ChatRouter
    FastAPI --> FeedbackRouter
    FastAPI --> HealthRouter
    
    AuthRouter --> SecretsManager
    ChatRouter --> ChatController
    FeedbackRouter --> HistoryService
    
    ChatController --> Dependencies
    ChatController --> ChatService
    ChatController --> HistoryService
    ChatController --> Guardrail
    
    Dependencies --> ChatService
    Dependencies --> HistoryService
    Dependencies --> BedrockClient
    
    ChatService --> BedrockClient
    ChatService --> Tools
    HistoryService --> DynamoDB
    StreamingService --> ChatService
    
    BedrockClient --> Bedrock
    Tools --> OpenSearch
    Guardrail --> Bedrock
    
    %% Cross-cutting connections
    Config -.-> ChatService
    Config -.-> HistoryService
    Config -.-> BedrockClient
    Logger -.-> ChatController
    Logger -.-> ChatService
    Models -.-> ChatRouter
    
    %% Styling
    classDef client fill:#e1f5fe
    classDef presentation fill:#f3e5f5
    classDef application fill:#e8f5e8
    classDef domain fill:#fff3e0
    classDef infrastructure fill:#fce4ec
    classDef external fill:#f1f8e9
    classDef crosscutting fill:#f5f5f5
    
    class Client client
    class AuthRouter,ChatRouter,FeedbackRouter,HealthRouter presentation
    class ChatController,Dependencies,Exceptions application
    class ChatService,HistoryService,StreamingService domain
    class BedrockClient,Guardrail,Tools infrastructure
    class Bedrock,DynamoDB,OpenSearch,SecretsManager external
    class Config,Logger,Models crosscutting
```

## 🔄 Request Flow Diagram

```mermaid
sequenceDiagram
    participant C as Client
    participant R as Chat Router
    participant CC as Chat Controller
    participant G as Guardrail
    participant CS as Chat Service
    participant BC as Bedrock Client
    participant H as History Service
    participant D as DynamoDB
    
    C->>R: POST /api/chat
    R->>G: check_input_guardrail()
    G-->>R: validation result
    R->>CC: handle_chat_request()
    CC->>H: get_messages()
    H->>D: load history
    D-->>H: chat history
    H-->>CC: loaded messages
    CC->>CS: generate_streaming_response()
    CS->>BC: LLM request
    BC-->>CS: streaming chunks
    CS-->>CC: formatted data
    CC->>H: save messages
    H->>D: store history
    CC-->>C: SSE stream
```

## 📊 Layer Responsibilities

| Layer | Components | Responsibilities |
|-------|------------|------------------|
| **Presentation** | Routers | API 엔드포인트, 요청 검증, 응답 형식화 |
| **Application** | Controller, DI | 비즈니스 플로우 오케스트레이션, 의존성 관리 |
| **Domain** | Services | 핵심 비즈니스 로직, 도메인 규칙 |
| **Infrastructure** | Clients, Tools | 외부 시스템 연동, 기술적 구현 |

## 🎯 Key Design Patterns

### **1. Dependency Injection Pattern**
```python
# dependencies.py
@lru_cache()
def get_bedrock_clients():
    return get_all_bedrock_clients()

def create_chat_service(model_id: str = None, group: str = "common"):
    return ChatService(model=model_id or config.model_id, ...)
```

### **2. Streaming Architecture Pattern**
```python
# ChatService: 순수 데이터 생성
async def generate_streaming_response() -> AsyncGenerator[Dict[str, Any], None]:
    yield {'role': 'assistant', 'content': content}

# Controller: SSE 형식 변환
async def stream_generator():
    async for item in chat_service.generate_streaming_response():
        if isinstance(item, dict):
            yield f"data: {json.dumps(item)}\\n\\n"
```

### **3. Error Handling Pattern**
```python
# exceptions.py
def create_http_exception(status_code: int, detail: str, error_type: str):
    return HTTPException(status_code=status_code, detail={
        "error_type": error_type,
        "message": detail
    })
```

## 🚀 Benefits

- **🔧 Maintainability**: 명확한 책임 분리로 유지보수 용이
- **🧪 Testability**: 의존성 주입으로 단위 테스트 가능
- **📈 Scalability**: 레이어별 독립적 확장 가능
- **🔄 Flexibility**: 인터페이스 기반 구현체 교체 용이
- **⚡ Performance**: 스트리밍 아키텍처로 실시간 응답

## 🔮 Future Enhancements

- [ ] **Middleware Pipeline**: 요청/응답 처리 파이프라인 확장
- [ ] **Caching Layer**: Redis 기반 캐싱 레이어 추가
- [ ] **Event Sourcing**: 도메인 이벤트 기반 아키텍처 도입
- [ ] **Circuit Breaker**: 외부 서비스 장애 대응 패턴 적용