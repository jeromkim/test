# 스트리밍 응답 아키텍처 비교

## 🏗️ 3가지 아키텍처 패턴

### **Pattern 1: 현재 방식 (Mixed Responsibility)**
```python
# LangChain에서 SSE 포맷 생성
async def generate_streaming_response(self):
    async for chunk in self.llm.astream(messages):
        if chunk.content:
            # ❌ 비즈니스 로직에서 프로토콜 포맷 생성
            yield f"data: {json.dumps({'role': 'assistant', 'content': content})}\n\n"

# FastAPI에서 중계
async def stream_generator():
    async for item in chat_service.generate_streaming_response():
        if isinstance(item, str):
            yield item  # SSE 문자열 그대로 전달
```

### **Pattern 2: 레이어 분리 (Recommended)**
```python
# LangChain: 순수 데이터만 반환
async def generate_streaming_response(self):
    async for chunk in self.llm.astream(messages):
        if chunk.content:
            # ✅ 순수 데이터 객체 반환
            yield StreamingChunk(
                type="content",
                role="assistant", 
                content=content,
                metadata={"message_id": ai_response_message_id}
            )

# FastAPI: 프로토콜 변환 담당
async def stream_generator():
    async for chunk in chat_service.generate_streaming_response():
        if chunk.type == "content":
            # ✅ 프레젠테이션 레이어에서 SSE 포맷 변환
            sse_data = f"data: {json.dumps(chunk.to_dict())}\n\n"
            yield sse_data
        elif chunk.type == "message":
            await history_service.add_message(chunk.message)
```

### **Pattern 3: 이벤트 기반 (Advanced)**
```python
# LangChain: 이벤트 발행
async def generate_streaming_response(self):
    async for chunk in self.llm.astream(messages):
        # ✅ 도메인 이벤트 발행
        event = ContentGeneratedEvent(
            content=content,
            session_id=session_id,
            message_id=message_id
        )
        await event_bus.publish(event)
        yield event

# FastAPI: 이벤트 구독 및 변환
class SSEEventHandler:
    async def handle_content_generated(self, event: ContentGeneratedEvent):
        sse_data = f"data: {json.dumps(event.to_sse_format())}\n\n"
        await self.send_to_client(sse_data)
```

## 📊 패턴별 비교 분석

### **Pattern 1 (현재): Mixed Responsibility**

#### 장점 ✅
- 구현이 간단하고 직관적
- 코드 라인 수가 적음
- 빠른 개발 가능

#### 단점 ❌
- **관심사 분리 위반**: 비즈니스 로직에 프로토콜 포맷 혼재
- **테스트 어려움**: SSE 포맷과 비즈니스 로직이 결합
- **재사용성 낮음**: 다른 프로토콜(WebSocket, gRPC) 지원 어려움
- **유지보수 어려움**: SSE 포맷 변경 시 비즈니스 로직 수정 필요

```python
# 문제 예시: 테스트 시 SSE 포맷까지 검증해야 함
async def test_chat_service():
    result = await chat_service.generate_streaming_response()
    assert "data: " in result  # 비즈니스 로직 테스트에 프로토콜 검증 포함
```

### **Pattern 2 (권장): 레이어 분리**

#### 장점 ✅
- **명확한 책임 분리**: 비즈니스 로직 vs 프레젠테이션 로직
- **테스트 용이성**: 각 레이어 독립적 테스트 가능
- **확장성**: 다른 프로토콜 쉽게 추가 가능
- **재사용성**: 비즈니스 로직을 다른 인터페이스에서 재사용

#### 단점 ❌
- 초기 구현 복잡도 증가
- 데이터 변환 오버헤드 (미미함)

```python
# 개선된 구조
class StreamingChunk:
    type: str  # "content", "tool_call", "message"
    role: str
    content: str
    metadata: dict
    
    def to_dict(self) -> dict:
        return {"role": self.role, "content": self.content}

# 비즈니스 로직 테스트
async def test_chat_service():
    chunks = []
    async for chunk in chat_service.generate_streaming_response():
        chunks.append(chunk)
    assert chunks[0].content == "expected_content"  # 순수 비즈니스 로직 테스트

# 프로토콜 테스트
def test_sse_formatter():
    chunk = StreamingChunk(type="content", role="assistant", content="test")
    sse_data = format_as_sse(chunk)
    assert sse_data == 'data: {"role": "assistant", "content": "test"}\n\n'
```

### **Pattern 3 (고급): 이벤트 기반**

#### 장점 ✅
- **완전한 디커플링**: 이벤트 기반 느슨한 결합
- **확장성**: 새로운 이벤트 핸들러 쉽게 추가
- **모니터링**: 이벤트 기반 로깅/메트릭 수집
- **마이크로서비스 준비**: 이벤트 기반 아키텍처

#### 단점 ❌
- 높은 복잡도
- 디버깅 어려움
- 성능 오버헤드

## 🎯 권장 아키텍처: Pattern 2

### **구현 예시**

```python
# 1. 데이터 모델 정의
from dataclasses import dataclass
from typing import Optional, Dict, Any

@dataclass
class StreamingChunk:
    type: str  # "content", "tool_result", "message", "error"
    role: Optional[str] = None
    content: Optional[str] = None
    tool_call_id: Optional[str] = None
    metadata: Optional[Dict[str, Any]] = None
    
    def to_dict(self) -> dict:
        result = {"type": self.type}
        if self.role:
            result["role"] = self.role
        if self.content:
            result["content"] = self.content
        if self.tool_call_id:
            result["tool_call_id"] = self.tool_call_id
        return result

# 2. 비즈니스 로직 (순수 데이터)
class ChatService:
    async def generate_streaming_response(self, messages, message_id, group):
        try:
            async for chunk in self.llm.astream(messages):
                if chunk.content:
                    yield StreamingChunk(
                        type="content",
                        role="assistant",
                        content=self._extract_content(chunk),
                        metadata={"message_id": message_id}
                    )
                
                # 도구 호출 처리
                if chunk.tool_calls:
                    for tool_call in chunk.tool_calls:
                        tool_result = await self._execute_tool(tool_call)
                        yield StreamingChunk(
                            type="tool_result",
                            tool_call_id=tool_call["id"],
                            content=tool_result,
                            metadata={"tool_name": tool_call["name"]}
                        )
                        
            # 완료 신호
            yield StreamingChunk(type="complete", metadata={"message_id": message_id})
            
        except Exception as e:
            yield StreamingChunk(type="error", content=str(e))

# 3. 프레젠테이션 로직 (SSE 변환)
class SSEFormatter:
    @staticmethod
    def format_chunk(chunk: StreamingChunk) -> str:
        return f"data: {json.dumps(chunk.to_dict(), ensure_ascii=False)}\n\n"
    
    @staticmethod
    def format_done() -> str:
        return "data: [DONE]\n\n"

# 4. FastAPI 컨트롤러
async def stream_generator(messages, message_id, group):
    formatter = SSEFormatter()
    
    async for chunk in chat_service.generate_streaming_response(messages, message_id, group):
        # 히스토리 저장 (사이드 이펙트)
        if chunk.type == "complete":
            await history_service.add_ai_message(chunk.metadata["message_id"])
        
        # SSE 포맷 변환
        sse_data = formatter.format_chunk(chunk)
        yield sse_data
    
    # 스트림 종료
    yield formatter.format_done()
```

## 🚀 마이그레이션 계획

### **Phase 1: 데이터 모델 도입**
```python
# StreamingChunk 클래스 추가
# 기존 코드와 병행 운영
```

### **Phase 2: 비즈니스 로직 분리**
```python
# ChatService에서 순수 데이터 반환하도록 수정
# SSE 포맷팅을 FastAPI로 이동
```

### **Phase 3: 테스트 및 최적화**
```python
# 각 레이어별 독립적 테스트 추가
# 성능 최적화 및 모니터링 강화
```

## 📋 결론

**Pattern 2 (레이어 분리)**가 가장 균형잡힌 선택입니다:

- ✅ **유지보수성**: 명확한 책임 분리
- ✅ **테스트 용이성**: 각 레이어 독립 테스트
- ✅ **확장성**: 다른 프로토콜 쉽게 추가
- ✅ **실용성**: 과도한 복잡도 없이 구현 가능

현재 시스템에서 점진적으로 마이그레이션하기에도 적합합니다!