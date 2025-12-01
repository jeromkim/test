# 코드베이스 마이그레이션 가이드

## 1. 파일 복사 매핑

### Backend (FastAPI)
```bash
# 기존 경로 → 새 경로
packages/app-fastapi/ → app/backend-fastapi/

# 구체적 복사 명령어
cp -r packages/app-fastapi/* app/backend-fastapi/
```

### Frontend (Streamlit)
```bash
# 기존 경로 → 새 경로  
packages/frontend-streamlit/ → app/frontend-streamlit/

# 구체적 복사 명령어
cp -r packages/frontend-streamlit/* app/frontend-streamlit/
```

## 2. 문서 생성

### ADR (Architecture Decision Records)
- `docs/adr/ADR-POC-V0.1.md`: POC 버전 결정사항
- `docs/adr/ADR-001-model-selection.md`: 모델 선택 근거
- `docs/adr/ADR-002-authentication.md`: JWT 인증 방식 선택

### 아키텍처 문서
- `architecture/diagram.png`: 시스템 아키텍처 다이어그램
- `architecture/cdk/`: AWS CDK 인프라 코드

## 3. 루트 파일들
- `README.md`: 프로젝트 전체 개요
- `.gitignore`: 통합 gitignore
- `docker-compose.yml`: 로컬 개발환경