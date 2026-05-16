# Development Log

## 2026-05-08

- 콘솔 새 글 쓰기 UI를 일반 사용자용 3단계 흐름으로 정리하고 프로젝트 ID 입력을 고급 옵션으로 이동
- 콘텐츠 유형/작성 방향/체험단 규칙을 워크플로 생성 요청에 포함하고 draft agent 프롬프트까지 전달
- 최종 결과물 편집본을 서버 artifact `edits/` 디렉터리에 저장/재조회하는 API와 콘솔 버튼 추가
- 작업 기록 상세에서 실패 워크플로를 바로 재시도하고 SSE/폴링으로 진행 상태를 추적하도록 개선
- 작업 기록 목록에 검색과 완료/실패 필터 추가
- 새 글쓰기 진행 중 새로고침해도 최신 워크플로를 다시 불러와 진행/결과 화면으로 복구
- 업로드 제한 조회 API(`GET /api/v1/uploads/config`)를 추가하고 콘솔 파일 검증/accept 속성을 서버 설정과 동기화
- 업로드 UX에서 용량/개수/형식/중복 파일 제외 사유를 사용자에게 표시
- 로그인 토큰은 기본적으로 sessionStorage에만 저장하고, 사용자가 선택할 때만 localStorage에 유지하도록 변경
- 작업 기록 목록/상세에서 워크플로 메타데이터 한 건만 삭제하는 API와 콘솔 액션 추가
- 완료된 워크플로의 `run` 재요청과 실패 워크플로의 `run`/`retry` 재진입을 API 테스트로 고정
- `photo_grouping_agent`의 전략별 의미 태그 필터와 의미 점수 가중치를 `strategy_profiles.py`로, boundary 판단을 `boundary_evaluators.py`로 분리하고 단위 테스트 추가
- 메타데이터가 부족한 사진은 파일명 숫자 순서가 크게 끊기는 경우에만 `filename_sequence_gap`으로 보수적으로 분리
- `photo_grouping_agent/scripts/compare-models.sh`를 추가하고 CLI 모델 비교 결과 저장 테스트를 보강
- 실제 `qwen2.5:14b`/`gemma4:e4b` 비교 결과를 생성하고, LLM이 일부 photo_id를 누락하는 경우 규칙 기반 그룹 조각으로 `coverage_repair`를 붙이도록 후처리 추가
- 프롬프트에 required photo_id 전체 커버 규칙을 추가한 뒤 `gemma4:e4b`는 전체 커버, `qwen2.5:14b`는 누락 2장을 자동 복구하는 결과를 확인
- 모델 비교 결과에 `quality_summary`를 추가하고, 재실행 기준 두 모델 모두 전체 커버/repair 없음/4개 그룹 결과를 확인
- 모델 비교 결과에 `recommended_model`을 추가하고 두 샘플 비교 결과를 저장. 샘플별 추천 모델이 갈려 추가 평가가 필요함을 확인
- LLM이 중첩 `groups` 같은 계약 외 필드를 반환해도 group 객체에서 허용 필드만 남기도록 schema repair 추가
- 여러 비교 결과를 집계하는 `model_comparison_report` 스크립트를 추가하고 현재 2개 샘플 기준 `gemma4:e4b` 추천 리포트 생성
- `compare-models.sh`가 입력 JSON의 `grouping_strategy`를 기본값으로 사용하도록 수정하고, 예제 입력 묶음 전체 비교/리포트 갱신용 `compare-sample-suite.sh` 추가
- suite 재실행 기준 현재 2개 샘플 집계 추천 모델은 `qwen2.5:14b`
- suite 샘플 manifest(`examples/model_comparison_samples.json`)를 추가하고 Ollama 비교 호출을 `temperature: 0`으로 고정
- 에이전트 HTTP 호출 공통 connect/read timeout, retry/backoff 설정과 Docker 환경변수 추가
- HTTP 에이전트 `/health` 응답에 `service` 필드를 표준화하고 core 검증 범위를 확대
- 결과물 수정본 latest 유지와 타임스탬프 버전 파일 보존 개수 제한 정책 추가
- `draft_agent` 테스트가 실제 Ollama를 호출하지 않게 고정해 검증 시간을 39초대에서 밀리초 단위로 단축
- `spring_orchestrator`, `momently_console`, `draft_agent`, `photo_exif_llm_pipeline` 검증 통과 확인
- `voice_profile_agent`에 공개 네이버 블로그 URL 샘플 추가 API를 붙이고, 콘솔 말투 학습 화면에서 URL만으로 본문 추출/말투 분석을 시작할 수 있게 함
- 말투 분석 Ollama 호출에 `VOICE_ANALYSIS_TEMPERATURE`, `VOICE_ANALYSIS_TOP_P` 옵션을 추가해 모델 파라미터를 환경변수로 조정 가능하게 함
- Docker 기준 확인용 `scripts/docker-up.sh`, `scripts/verify-docker-stack.sh`를 추가해 compose rebuild/up 후 콘솔 정적 앱, 로그인, 인증 API, 말투 프로필/URL 학습 라우트 smoke test를 한 번에 실행
- 말투 샘플 추가 요청에 `fast_analysis` 옵션을 추가하고 Docker smoke test에서 실제 샘플 저장/프로필 통계 갱신까지 빠르게 확인
- `style_agent`에 `deterministic_voice` 옵션을 추가하고 Docker smoke test가 학습된 voice profile을 문체 적용 단계까지 넘겨 확인하도록 확장
- `VOICE_BLOG_IMPORT_FIXTURE_MAP`과 네이버 블로그 fixture HTML을 추가해 Docker smoke test가 외부 네트워크 없이 URL 본문 추출/학습까지 검증

## 2026-05-05

- Docker compose 환경에서 실제 업로드 API로 MP4 동영상을 올리고 전체 워크플로가 `COMPLETED`까지 도달하는 E2E 검증 완료
- 실패 워크플로 재실행 시 정상 단계로 재진입하면 `lastFailedStep`, `lastErrorMessage`를 지우도록 도메인 규칙 보강
- PostgreSQL 저장소 Testcontainers 통합 테스트를 추가하고 `RUN_POSTGRES_INTEGRATION_TESTS=true` opt-in 방식으로 정리
- `photo_grouping_agent` LLM 보정 프롬프트를 전체 사진 목록 대신 그룹 후보와 필요한 사진 요약 중심으로 축약
- `spring_orchestrator`, `momently_console`, `photo_exif_llm_pipeline`, `photo_grouping_agent` 표준 검증 통과 확인
- Docker compose에 Python 에이전트, orchestrator, gateway healthcheck를 추가하고 orchestrator/gateway 기동 조건을 `service_healthy` 기준으로 강화
- `POST /api/v1/workflows/{id}/retry`를 추가하고 `run`/`retry` 중복 요청을 멱등 응답으로 정리
- 콘솔 실패 화면에 재시도 버튼을 연결하고, 실패한 수동 워크플로는 `retry` 엔드포인트를 호출하도록 변경
- 사진/동영상 입력 폴더 누락 실패 메시지에서 내부 컨테이너 경로를 제거하고 사용자 행동 중심 문장으로 정리
- 사진 분석과 동영상 대표 프레임 분석을 제한된 worker pool로 병렬 처리하도록 정리
- Python CLI `--analysis-concurrency`, Spring `agents.photo-info.pipeline.analysis-concurrency`, Docker `PHOTO_PIPELINE_ANALYSIS_CONCURRENCY` 설정 연결

## 2026-05-04

- `spring_orchestrator` PostgreSQL persistence adapter의 도메인/JPA 매핑 및 repository 위임 단위 테스트 추가
- 표준 검증 명령 `gradle test jacocoTestReport jacocoTestCoverageVerification` 통과 확인
- 업로드 API를 사진 전용에서 미디어 업로드로 확장하고 `mp4`, `mov`, `m4v` 동영상 저장 지원 추가
- 콘솔 파일 업로드 UI에서 사진과 동영상을 함께 선택할 수 있도록 확장
- `photo_exif_llm_pipeline`에 동영상 스캔, `ffmpeg` 대표 프레임 추출, 기존 비전 모델 기반 동영상 요약 흐름 추가
- 동영상 분석을 단일 대표 프레임에서 여러 대표 프레임 샘플링/요약 병합 방식으로 확장
- Spring 오케스트레이터와 Docker 설정에 동영상 대표 프레임 샘플링 CLI 옵션 연결
- 워크플로 실행/문체 재적용 진행 상태를 SSE 우선, 폴링 fallback 방식으로 갱신

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

## 2026-05-17

### 초대 코드 회원가입

- Spring 오케스트레이터에 `POST /api/v1/auth/register` 추가
- `MOMENTLY_SIGNUP_INVITE_CODE`가 비어 있으면 회원가입을 비활성화하고, 설정된 경우 초대 코드 검증 후 BCrypt 해시로 사용자 계정 저장
- 기존 환경변수 콘솔 계정 로그인은 유지하되, DB 사용자 로그인도 함께 지원
- 콘솔 로그인 화면에 로그인/회원가입 전환 UI와 초대 코드 입력 추가
- Docker smoke test에 회원가입 후 보호 API 접근 검증 추가
