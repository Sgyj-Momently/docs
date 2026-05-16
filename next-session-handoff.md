# Next Session Handoff

## 목적

이 문서는 다른 PC나 다른 세션에서 바로 다음 작업을 이어갈 수 있도록 현재 상태와 시작 순서를 요약한다.

## 먼저 읽을 문서

1. [Agent.md](../Agent.md)
2. [spring_orchestrator/AGENT.md](../spring_orchestrator/AGENT.md)
3. [project-overview.md](./project-overview.md)
4. [orchestrator-design.md](./orchestrator-design.md)
5. [adr/003-id-strategy.md](./adr/003-id-strategy.md)
6. [adr/004-database-strategy.md](./adr/004-database-strategy.md)

## 현재 완료 상태

### photo_exif_llm_pipeline

- 폴더 단위 이미지/동영상 스캔 가능
- EXIF 추출 가능
- 동영상은 `ffmpeg`로 여러 대표 프레임 추출 가능
- Ollama 비전 모델 기반 이미지/대표 프레임 요약 가능
- bundle JSON 생성 가능
- writer model과 vision model 분리 가능
- 동영상 대표 프레임 요약을 동영상 단위 요약으로 병합 가능

### photo_grouping_agent

- 그룹화 전략 enum 기반 입력 계약 존재
- FastAPI 진입점 존재
- `/docs`, `/openapi.json` 확인 가능
- 공개 API에서 `ollama_base_url`, `ollama_timeout_seconds` 제거 완료
- 전략별 의미 태그 필터와 의미 점수 가중치는 `strategy_profiles.py`, boundary 판단은 `boundary_evaluators.py`로 분리
- 메타데이터가 부족해도 파일명 숫자 순서가 크게 끊기면 `filename_sequence_gap`으로 보수적으로 분리
- `scripts/compare-models.sh`로 `qwen2.5`/`gemma4` 그룹화 비교 결과를 같은 입력 기준으로 저장 가능
- LLM 보정 결과가 입력 `photo_id`를 누락하면 규칙 기반 그룹 조각으로 `coverage_repair`를 붙이고, 중복/추가 ID는 `invalid_group_coverage`로 처리
- 모델 비교 결과에는 `quality_summary`가 포함되어 커버리지, repair 수, 규칙 기반 그룹 수 대비 차이를 한눈에 볼 수 있음
- 실제 `qwen2.5:14b`/`gemma4:e4b` 비교 결과는 `examples/grouped_compare_qwen_vs_gemma4.json`, `examples/grouped_compare_qwen_vs_gemma4_v2.json`에 저장됨
- 현재 샘플별 `recommended_model`은 1번 샘플 `gemma4:e4b`, 2번 샘플 `qwen2.5:14b`로 갈리므로 더 많은 샘플 비교가 필요
- `scripts/report-model-comparisons.sh`로 여러 비교 결과를 집계하며 현재 집계 리포트는 `examples/model_comparison_report.md`/`.json`에 저장됨
- `scripts/compare-sample-suite.sh`로 예제 입력 묶음 전체 비교와 집계 리포트 갱신을 한 번에 실행 가능
- suite 샘플 목록은 `examples/model_comparison_samples.json`에서 관리
- `compare-models.sh`는 입력 JSON의 `grouping_strategy`를 기본값으로 사용하고, 없으면 `LOCATION_BASED`를 사용
- Ollama 비교 호출은 재현성을 위해 `temperature: 0`으로 실행
- 현재 2개 샘플 suite 집계 기준 추천 모델은 `qwen2.5:14b`

### spring_orchestrator

- 헥사고날 구조 초안 존재
- `WorkflowController`, `WorkflowService`, `WorkflowStateMachine`, `WorkflowRunner` 구현됨
- `memory` 프로필 저장소와 `postgres` 프로필 JPA adapter 존재
- `/api/v1/uploads/media`로 사진/동영상 업로드 후 프로젝트 ID 생성 가능
- `/api/v1/uploads/config`로 현재 업로드 제한과 지원 확장자 조회 가능
- `local-photo-info` 실행 시 동영상 프레임 샘플링 옵션을 CLI로 전달
- Docker compose 환경에서 업로드된 MP4 동영상 워크플로가 `COMPLETED`까지 도달함을 확인
- 워크플로 실행/문체 재적용 진행 상태는 SSE 우선, 폴링 fallback 방식으로 갱신
- FAILED 재실행 시 정상 단계로 재진입하면 실패 메타데이터를 지움
- `POST /api/v1/workflows/{workflowId}/retry`로 명시 재시도 가능
- `run`/`retry` 중복 요청은 멱등 응답으로 처리
- 완료된 워크플로의 `run` 재요청과 실패 워크플로의 `run`/`retry` 재진입 계약을 API 테스트로 고정
- 입력 묶음 누락 같은 사용자-facing 실패 메시지는 내부 컨테이너 경로를 노출하지 않음
- FastAPI 에이전트 HTTP 호출에는 공통 connect/read timeout과 retry/backoff 설정이 적용됨
- HTTP 에이전트 `/health` 응답은 `status`와 `service` 필드를 공통으로 포함함
- 테스트 및 JaCoCo 커버리지 검증 통과
- PostgreSQL 저장소 Testcontainers 통합 테스트가 있으며 기본 검증에서는 skip, `RUN_POSTGRES_INTEGRATION_TESTS=true`로 opt-in 실행

### momently_console

- 프로젝트 ID 입력 모드와 사진/동영상 직접 업로드 모드가 공존
- 업로드 모드는 서버가 생성한 프로젝트 ID로 워크플로를 생성/실행
- 서버 업로드 정책을 읽어 개수/용량/확장자/중복 파일을 사전 검증
- 워크플로 기록 목록/검색/상태 필터/개별 삭제/전체 삭제와 결과 아티팩트 확인 가능
- 실패한 워크플로 화면과 작업 기록 상세에서 재시도 가능
- 완료 결과물은 콘솔에서 편집하고 서버 저장본으로 남길 수 있음
- 결과물 수정본은 latest 파일을 유지하고, 타임스탬프 버전 파일은 설정 개수만 보존함
- 새 글쓰기 화면은 세션에 남은 최신 워크플로 ID를 기준으로 새로고침 후 진행/결과 화면을 복구함
- 로그인 토큰은 기본 세션 저장이며, 사용자가 선택할 때만 브라우저 유지

## 스프링 표준 검증 명령

핵심 모듈 전체를 한 번에 보려면:

```bash
./scripts/verify-core.sh
```

현재 환경 차이를 줄이기 위해 아래 명령을 표준으로 사용한다.

```bash
cd spring_orchestrator
env GRADLE_USER_HOME=.gradle-home GRADLE_OPTS='-Dorg.gradle.native=false' gradle test jacocoTestReport jacocoTestCoverageVerification
```

PostgreSQL Testcontainers 통합 테스트까지 실행하려면 Docker/Testcontainers 환경을 먼저 맞춘 뒤 아래처럼 실행한다.

```bash
cd spring_orchestrator
RUN_POSTGRES_INTEGRATION_TESTS=true env GRADLE_USER_HOME=.gradle-home GRADLE_OPTS='-Dorg.gradle.native=false' gradle test
```

## 현재 기준 의사결정

- 공개 워크플로 식별자 기본 전략: `UUIDv7`
- 운영 메타데이터 DB 기본 전략: `PostgreSQL`
- 대용량 산출물은 DB 본문이 아니라 artifact 저장소에 분리
- 전체 파이프라인 순서와 상태 머신은 Spring 오케스트레이터가 관리
- 에이전트는 자기 단계 입력을 받아 결과만 반환

## 바로 다음 우선 작업

### 1. spring_orchestrator

- Testcontainers가 Docker Desktop 29 소켓을 안정적으로 잡도록 CI/로컬 실행 환경 정리
- 운영 schema migration 전략 결정

### 2. photo_grouping_agent

- 실제 사용자 샘플을 추가해 `model_comparison_report` 신뢰도 높이기

### 3. 운영/UX 검증

- SSE 재연결/폴링 fallback을 브라우저 E2E로 검증
- 에이전트별 헬스 체크와 장애 메시지 표준화

## 작업 시작 체크리스트

- Java 25 설치 확인
- ffmpeg 설치 확인
- Ollama 실행 여부 확인
- 필요한 모델(`qwen2.5vl:7b`, `qwen2.5:14b`, 필요 시 `gemma4`) 존재 확인
- `spring_orchestrator` 테스트 먼저 통과 확인
- `./scripts/verify-core.sh`로 Spring, 콘솔, 핵심 에이전트, 사진/동영상 파이프라인 영향 확인
- 새 기능 추가 전 관련 테스트부터 작성

## 주의 사항

- 테스트가 실제로 통과하지 않은 상태를 완료로 간주하지 않는다.
- 스프링 쪽은 웹 DTO가 application 계층으로 새지 않도록 유지한다.
- 공개 API에는 인프라 설정값을 노출하지 않는다.
- 문서, ADR, 개발 일지는 코드 변경과 함께 갱신한다.
