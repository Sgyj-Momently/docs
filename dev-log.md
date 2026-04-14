# Development Log

## 2026-04-16

- `photo_exif_llm_pipeline` 초기 구조 생성
- EXIF 추출, 사진별 요약, bundle 생성 흐름 추가
- TDD 기반 테스트 구조 도입
- 주석과 docstring을 한글 기준으로 정리

## 2026-04-20

### 구조 및 원칙

- 에이전트별 모듈 분리 방향을 명확히 함
- 상위 워크스페이스는 유지하되, 각 에이전트는 독립 계약과 책임을 가지도록 정리
- `Agent.md`에 에이전트 모듈 원칙 추가
- `Agent.md`에 API 설계 원칙 추가

### photo_grouping_agent 생성

- `photo_grouping_agent` 모듈 생성
- 입력/출력 JSON 스키마 추가
- 예제 입력/출력 파일 추가
- 오케스트레이션 메모 추가

### 그룹화 로직

- 시간 기반 1차 그룹화 구현
- `captured_at`이 없을 때 `location_hint`, `scene_type`, `summary`를 활용하는 규칙 추가
- `group_reason`, `score`, `score_details` 필드 추가
- beach/urban 충돌 태그 기반 분리 규칙 추가

### 전략 기반 그룹화

- `GroupingStrategy` enum 도입
- 허용 전략:
  - `TIME_BASED`
  - `LOCATION_BASED`
  - `SCENE_BASED`
  - `FOOD_TYPE_BASED`
  - `STORY_FLOW_BASED`
- 자유 텍스트 대신 enum 전략만 받는 구조로 정리

### API / 문서

- 그룹화 API 문서 작성
- OpenAPI YAML 초안 작성
- FastAPI 서버 진입점 추가
- Swagger UI(`/docs`)와 OpenAPI(`/openapi.json`) 확인

### 공개 API 경계 정리

- `ollama_base_url`과 `ollama_timeout_seconds`를 공개 요청에서 제거
- 해당 값들을 서버 내부 환경 설정으로 이동
- 공개 API에는 도메인 의도와 데이터만 남기기로 결정

### Spring 오케스트레이터 방향 정리

- Spring이 전체 파이프라인 순서와 상태 머신을 관리하는 구조로 방향 확정
- `WorkflowStateMachine`, `WorkflowRunner` 중심의 순차 실행 뼈대 추가
- 에이전트 호출은 outbound port/adapter를 통해서만 수행하도록 정리

### 인프라 전략 의사결정

- 외부 공개 식별자와 워크플로 식별자의 기본 전략을 `UUIDv7`로 정리
- 기본 운영 DB를 `PostgreSQL`로 정리
- 대용량 JSON 산출물은 DB 본문이 아니라 artifact 저장소로 분리하기로 결정
- 높은 트래픽과 분산 가능성을 기본 가정으로 문서와 규칙을 보강

### 문서화 작업

- 전역 `Agent.md`에 문서화 원칙 추가
- `spring_orchestrator/AGENT.md`에 ID 전략, DB 전략 원칙 추가
- ADR `003`, `004` 추가
- `docs/orchestrator-design.md`에 PostgreSQL, UUIDv7, artifact 분리 원칙 반영

### 테스트 및 인수인계 정리

- Spring 오케스트레이터 JaCoCo 커버리지 검증 추가
- 기존 기능 기준 테스트 커버리지 90% 이상 확인
- 다른 PC에서도 이어서 작업할 수 있도록 세션 인수인계 문서 추가
- `spring_orchestrator/README.md`, `docs/project-overview.md`, `docs/roadmap.md`에 현재 상태와 다음 작업 반영

### 현재 상태

- `photo_grouping_agent` 테스트 통과
- FastAPI `/docs` 사용 가능
- `photo_exif_llm_pipeline -> photo_grouping_agent` 어댑터 연결 완료

## 남은 메모

- `qwen2.5:14b` 기반 LLM 보정은 아직 느린 편
- `gemma4`는 설치 완료 후 비교 실험 예정
- 그룹화 규칙은 더 전략별로 분리할 여지가 있음
