# 함수별 상세 매핑

## 🔐 Authentication Layer

### `src/auth.py`
- `get_jwt_secret()`: AWS Secrets Manager에서 JWT 시크릿 조회
- `verify_token(req: TokenRequest)`: JWT 토큰 검증 및 사용자 정보 반환

### `app.py` (Frontend)
- `verify_token(token: str)`: FastAPI 인증 엔드포인트 호출
- `update_last_active()`: 사용자 활동 시간 업데이트

## 💬 Chat Processing Layer

### `src/services/chat_service.py`
- `__init__()`: LLM 클라이언트 초기화, 도구 바인딩
- `build_messages()`: 시스템 프롬프트 + 히스토리 + 사용자 메시지 구성
- `generate_streaming_response()`: 스트리밍 응답 생성, 도구 호출 처리
- `generate_complete_response()`: 일반 응답 생성

### `src/adapters/chat_controller.py`
- `handle_chat_request()`: 채팅 요청 전체 플로우 관리
- 히스토리 로드 → 사용자 메시지 저장 → LLM 호출 → 응답 스트리밍

## 🔍 Knowledge Retrieval Layer

### `src/tools/retrieve_knowledge_search.py`
- `retrieve_knowledge_base_search()`: 메인 검색 함수
- `rerank_results()`: Cohere 모델로 결과 재순위화
- `apply_dynamic_truncation()`: 점수별 문서 길이 최적화
- `truncate_document()`: 문단 단위 문서 자르기

## 📚 History Management Layer

### `src/services/history_service.py`
- `get_messages()`: DynamoDB에서 채팅 히스토리 조회
- `add_message()`: 메시지 저장 (BaseMessage 객체)
- `add_user_message()`: 사용자 메시지 저장
- `add_ai_message()`: AI 응답 저장
- `add_feedback_to_message()`: 특정 메시지에 피드백 추가
- `clear_history()`: 세션 히스토리 삭제

## 🛡️ Security & Validation Layer

### `main.py`
- `check_input_guardrail()`: Bedrock Guardrail로 입력 검증
- `create_guardrail_blocked_response()`: 차단 시 SSE 응답 생성
- `get_client_ip()`: 클라이언트 IP 추출

## 🌐 Frontend Communication Layer

### `utils/request.py`
- `call_fastapi_predict()`: 스트리밍 채팅 API 호출
- `send_feedback()`: 피드백 전송 API 호출

### `pages/chat.py`
- `generate_session_id()`: 세션 ID 생성
- `extract_source_info()`: 검색 결과에서 파일 정보 추출
- `get_chatbot_description()`: 챗봇 설명 동적 생성
- `feedback_modal()`: 피드백 모달 UI

## ⚙️ Configuration & Utilities

### `src/config.py`
- 환경변수 로드 및 설정 객체 생성
- AWS, OpenSearch, Knowledge Base 설정 관리

### `src/utils/logger.py`
- `setup_logger()`: 구조화된 로거 설정
- 개발/운영 환경별 로그 포맷 분기

### `src/utils/models.py`
- `ChatRequest`: 채팅 요청 모델
- `ChatResponse`: 채팅 응답 모델
- `SourceMetadata`: 소스 문서 메타데이터

## 🔄 데이터 플로우 함수 체인

### 인증 플로우
```
Portal JWT → verify_token() → get_jwt_secret() → update_last_active()
```

### 채팅 플로우
```
User Input → handle_chat_request() → build_messages() → 
generate_streaming_response() → retrieve_knowledge_base_search() → 
rerank_results() → apply_dynamic_truncation()
```

### 히스토리 플로우
```
get_messages() → add_user_message() → add_ai_message() → 
add_feedback_to_message()
```