# GenAI Chatbot Platform - Application Architecture

## 🏗️ 전체 시스템 구성도

```mermaid
graph TB
    %% External Systems
    Portal[MIDAS Portal] --> |JWT Token| Auth[JWT Authentication]
    
    %% Frontend Layer
    subgraph "Frontend (Streamlit)"
        App[app.py]
        Pages[pages/chat.py]
        Utils[utils/request.py]
        
        App --> |Token Verification| Auth
        App --> |Navigation| Pages
        Pages --> |API Calls| Utils
    end
    
    %% Backend Layer  
    subgraph "Backend (FastAPI)"
        Main[main.py]
        AuthRouter[src/auth.py]
        ChatController[src/adapters/chat_controller.py]
        ChatService[src/services/chat_service.py]
        HistoryService[src/services/history_service.py]
        KnowledgeSearch[src/tools/retrieve_knowledge_search.py]
        Config[src/config.py]
        Models[src/utils/models.py]
        Logger[src/utils/logger.py]
        
        Main --> |Include Router| AuthRouter
        Main --> |Handle Requests| ChatController
        ChatController --> |Business Logic| ChatService
        ChatController --> |History Management| HistoryService
        ChatService --> |Knowledge Retrieval| KnowledgeSearch
        ChatService --> |System Prompts| Prompts[src/prompts/chat.py]
    end
    
    %% AWS Services
    subgraph "AWS Services"
        Bedrock[Amazon Bedrock]
        SecretsManager[AWS Secrets Manager]
        DynamoDB[Amazon DynamoDB]
        KnowledgeBase[Bedrock Knowledge Base]
        Guardrail[Bedrock Guardrail]
        
        ChatService --> |LLM Calls| Bedrock
        AuthRouter --> |JWT Secret| SecretsManager
        HistoryService --> |Chat History| DynamoDB
        KnowledgeSearch --> |Document Search| KnowledgeBase
        Main --> |Content Filter| Guardrail
    end
    
    %% Data Flow
    Utils --> |HTTP Requests| Main
    Auth --> |Token Validation| AuthRouter
```

## 🔧 주요 컴포넌트 상세

### Frontend Components
- **app.py**: 메인 애플리케이션, JWT 인증 플로우
- **pages/chat.py**: 채팅 인터페이스, 파일 미리보기
- **utils/request.py**: API 통신, 스트리밍 처리

### Backend Components  
- **main.py**: FastAPI 앱, 라우터 등록, Guardrail
- **src/auth.py**: JWT 토큰 검증
- **src/adapters/chat_controller.py**: 요청 처리, 응답 스트리밍
- **src/services/chat_service.py**: LLM 서비스, 도구 호출
- **src/services/history_service.py**: 채팅 히스토리 관리
- **src/tools/retrieve_knowledge_search.py**: 지식 검색, 리랭킹

## 📊 함수 레벨 구성도

```mermaid
graph LR
    %% Authentication Flow
    subgraph "Authentication"
        verify_token[verify_token]
        get_jwt_secret[get_jwt_secret]
        update_last_active[update_last_active]
    end
    
    %% Chat Flow
    subgraph "Chat Processing"
        handle_chat_request[handle_chat_request]
        generate_streaming_response[generate_streaming_response]
        build_messages[build_messages]
        call_fastapi_predict[call_fastapi_predict]
    end
    
    %% Knowledge Retrieval
    subgraph "Knowledge Search"
        retrieve_knowledge_base_search[retrieve_knowledge_base_search]
        rerank_results[rerank_results]
        apply_dynamic_truncation[apply_dynamic_truncation]
    end
    
    %% History Management
    subgraph "History"
        get_messages[get_messages]
        add_message[add_message]
        add_feedback_to_message[add_feedback_to_message]
    end
    
    %% Guardrail
    subgraph "Security"
        check_input_guardrail[check_input_guardrail]
        create_guardrail_blocked_response[create_guardrail_blocked_response]
    end
```

## 🔄 데이터 플로우

1. **인증**: Portal → JWT Token → verify_token → Session
2. **채팅**: User Input → Guardrail → LLM → Knowledge Search → Response
3. **히스토리**: Messages → DynamoDB → Session Management
4. **피드백**: User Feedback → add_feedback_to_message → DynamoDB

## 🛡️ 보안 레이어

- JWT 토큰 검증 (AWS Secrets Manager)
- Bedrock Guardrail 콘텐츠 필터링
- 세션 타임아웃 관리
- CORS 정책 적용