# Next Session Handoff

## 목적

이 문서는 다른 PC나 다른 세션에서 바로 다음 작업을 이어갈 수 있도록 현재 상태와 시작 순서를 요약한다.

## 발행 패키지 포맷 (네이버 에디터 붙여넣기용)

콘솔의 `MetaSelectionCard` "발행 패키지 복사" 버튼이 만드는 클립보드
문자열의 정형 포맷. `momently_console/src/publicationPackage.js` 의
`buildPublicationPackage` 가 생성한다.

```
{선택된 제목 한 줄}

> {편집된 메타 디스크립션 (없으면 인용 블록 생략)}

{본문 마크다운 (선두 H1 은 제거 — 제목으로 대체됨)}

#태그1 #태그2 #태그3
```

- 해시태그는 `#` 접두 자동 부여, 중복·내부 공백 제거.
- 본문 안에 `# 제목` 형태의 H1 이 있으면 1회 제거(사용자가 고른 제목과 중복 방지).
- 끝에 줄바꿈 한 번을 강제(에디터 호환).
- 메타 디스크립션은 인용(`> ...`) 블록으로 노출해 시각 구분.

## 먼저 읽을 문서

1. [Agent.md](../Agent.md)
2. [spring_orchestrator/AGENT.md](../spring_orchestrator/AGENT.md)
3. [project-overview.md](./project-overview.md)
4. [orchestrator-design.md](./orchestrator-design.md)
5. [adr/003-id-strategy.md](./adr/003-id-strategy.md)
6. [adr/004-database-strategy.md](./adr/004-database-strategy.md)
7. [adr/005-agent-error-format.md](./adr/005-agent-error-format.md)

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
- `model_comparison_report`는 전략별 커버리지와 `confidence_level`을 표시한다. 현재 suite는 샘플 수와 전략 다양성이 부족해 `low` confidence 경고가 뜸

### spring_orchestrator

- 헥사고날 구조 초안 존재
- `WorkflowController`, `WorkflowService`, `WorkflowStateMachine`, `WorkflowRunner` 구현됨
- `memory` 프로필 저장소와 `postgres` 프로필 JPA adapter 존재
- `/api/v1/uploads/media`로 사진/동영상 업로드 후 프로젝트 ID 생성 가능
- `/api/v1/uploads/config`로 현재 업로드 제한과 지원 확장자 조회 가능
- `local-photo-info` 실행 시 동영상 프레임 샘플링 옵션을 CLI로 전달
- Docker compose 환경에서 업로드된 MP4 동영상 워크플로가 `COMPLETED`까지 도달함을 확인
- 워크플로 실행/문체 재적용 진행 상태는 SSE 우선, 폴링 fallback 방식으로 갱신
- 글쓰기 화면의 SSE 실패 -> 폴링 fallback 전환은 `workflowLiveUpdates.js` 순수 유틸과 Vitest로 고정
- FAILED 재실행 시 정상 단계로 재진입하면 실패 메타데이터를 지움
- `POST /api/v1/workflows/{workflowId}/retry`로 명시 재시도 가능
- `run`/`retry` 중복 요청은 멱등 응답으로 처리
- 완료된 워크플로의 `run` 재요청과 실패 워크플로의 `run`/`retry` 재진입 계약을 API 테스트로 고정
- 입력 묶음 누락 같은 사용자-facing 실패 메시지는 내부 컨테이너 경로를 노출하지 않음
- FastAPI 에이전트 HTTP 호출에는 공통 connect/read timeout과 retry/backoff 설정이 적용됨
- HTTP 에이전트 `/health` 응답은 `status`와 `service` 필드를 공통으로 포함함
- 테스트 및 JaCoCo 커버리지 검증 통과
- PostgreSQL 저장소 Testcontainers 통합 테스트가 있으며 기본 검증에서는 skip, `RUN_POSTGRES_INTEGRATION_TESTS=true`로 opt-in 실행
- voice profile 본문은 `VoiceProfileAgentClient` 가 `GET /api/v1/internal/voice-profiles/{id}?owner=...` 로 가져온다. Style/Draft client 가 `workflow.getOwnerUsername()` 을 같이 전달해 owner-scoped 검색을 수행한다. 로컬 fs fallback 은 제거됨. `agents.voice-profile.internal-token` (env: `MOMENTLY_INTERNAL_TOKEN`) 미설정 시 client 호출 자체를 건너뛰는 fail-closed
- Flyway V5 가 `workflows_status_check` 를 enum 전체로 재정의해 메타 단계 진입을 허용한다. `WorkflowStatus` 에 새 값을 추가할 때마다 같은 PR 에서 이 CHECK 도 갱신해야 한다

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
- 말투 학습 화면에서 공개 네이버 블로그 URL을 넣으면 `voice_profile_agent`가 본문을 추출해 샘플로 학습 가능
- 로컬 확인은 `./scripts/docker-up.sh`로 Docker compose 이미지를 rebuild/up한 뒤 콘솔 정적 앱, 로그인, 인증 API, 말투 프로필/URL 학습 라우트 smoke test까지 돌리는 흐름을 표준으로 사용
- PostgreSQL 프로필은 Flyway `V1__baseline_schema.sql`을 적용한 뒤 Hibernate `ddl-auto=validate`로 schema를 검증함. 기존 테이블이 있는 DB도 baseline version `0`으로 Flyway 이력을 붙인 뒤 V1 migration을 실행하도록 설정
- Testcontainers PostgreSQL 통합 테스트는 Docker Desktop 29 계열에서 Docker API 협상을 위해 `src/test/resources/docker-java.properties`의 `api.version=1.44`를 사용
- 회원가입은 로그인한 사용자가 콘솔 `초대 코드` 메뉴 또는 `POST /api/v1/auth/invites`로 1회용 코드를 먼저 발행한 뒤 진행하는 방식. `MOMENTLY_SIGNUP_INVITE_CODE`는 초기 부트스트랩 fallback으로만 유지
- 초대 코드 화면은 최근 발행 이력, 상태(`ACTIVE`, `USED`, `EXPIRED`, `REVOKED`), 사용 전 폐기를 지원
- 계정 화면은 현재 로그인 계정 출처를 보여주고, 초대 코드로 가입한 DB 계정의 비밀번호 변경을 지원. 초기 환경변수 콘솔 계정은 API 변경 불가
- 계정 화면은 관리자 기준 사용자 목록과 가입 사용자 활성/비활성 전환을 지원. 비활성화 사용자는 새 로그인이 차단되고, 비밀번호 변경/비활성화/재활성화 시 기존 JWT도 토큰 버전으로 즉시 무효화됨
- Docker 확인 스크립트는 로그인 토큰으로 초대 코드 발행/목록/폐기를 확인한 뒤, 별도 코드로 회원가입하고 보호 API 접근·현재 계정 조회·비밀번호 변경 후 재로그인·사용자 비활성화/활성화·기존 JWT 무효화까지 검증
- Docker smoke test는 말투 샘플 빠른 학습 결과를 style agent의 `deterministic_voice` 적용까지 넘겨 실제 문체 적용 경로도 확인
- Docker smoke test는 `VOICE_BLOG_IMPORT_FIXTURE_MAP` 기본값으로 fixture 기반 네이버 블로그 URL 본문 추출/학습도 확인

### voice_profile_agent

- 말투 샘플은 직접 붙여넣기와 공개 네이버 블로그 URL 입력을 모두 지원
- 네이버 블로그 URL은 모바일 글 URL로 정규화한 뒤 본문 컨테이너 텍스트를 추출
- Ollama 말투 분석/예시 생성 호출은 `VOICE_ANALYSIS_MODEL`, `VOICE_ANALYSIS_TEMPERATURE`, `VOICE_ANALYSIS_TOP_P`로 조정 가능
- 샘플 추가 요청에 `fast_analysis: true`를 넣으면 해당 요청은 Ollama 심층 분석 없이 로컬 통계만 갱신하므로 Docker smoke test에서 사용
- `VOICE_BLOG_IMPORT_FIXTURE_MAP`은 정규화된 네이버 블로그 URL을 `voice_profile_agent/src` 내부 HTML fixture 경로로 매핑하는 JSON 객체이며, Docker smoke에서 외부 네트워크 없이 URL 학습을 검증하는 용도
- 저장 백엔드는 `VOICE_PROFILE_STORAGE_BACKEND=minio` 로 MinIO 사용. 사용자 JWT 없이 서버 간 호출이 필요한 orchestrator 를 위해 `GET /api/v1/internal/voice-profiles/{id}?owner=...` 가 따로 있다. `X-Internal-Token` 헤더로 검증하고 `MOMENTLY_INTERNAL_TOKEN` env 가 비어 있으면 default-deny. gateway nginx 는 `^~ /api/v1/internal/` 를 404 로 차단해 외부 노출도 막음

### style_agent

- `deterministic_voice: true`를 요청에 넣으면 Ollama 재작성 없이 저장된 voice profile 특징으로 빠른 문체 적용을 수행
- 이 옵션은 Docker smoke test와 빠른 로컬 확인용이며 기본 글쓰기 흐름은 기존처럼 LLM 재작성 우선

### writing latency

- Docker 기본 글쓰기 tail 모델은 로컬 응답성을 우선해 `DRAFT_MODEL=qwen2.5:14b`, `STYLE_MODEL=qwen2.5:14b`, `REVIEW_MODEL=qwen2.5:14b`
- `review_agent`의 최종 LLM 교정은 `REVIEW_ENABLE_LLM_POLISH=false`가 기본이다. 고품질 최종 교정을 원할 때만 `true`로 켠다
- Docker 기본 비디오 분석은 빠른 초안 확인을 위해 `PHOTO_PIPELINE_VIDEO_FRAME_COUNT=1`로 둔다. 동영상 맥락 품질을 우선하면 `3` 이상으로 올린다
- `WorkflowRunner`는 각 단계별 `workflow_step_timing` 로그를 남겨 photo_info/draft/style/review 중 어디가 느린지 바로 확인할 수 있다
- 더 느려도 품질을 우선할 때는 `deploy/.env`에서 draft/style/review 모델을 `qwen2.5:32b`로 올릴 수 있다

### writing quality

- 말투 학습은 네이버 블로그 HTML/마크다운 샘플에 섞인 이미지 URL, 추적 URL, 지도/공유 UI 텍스트를 제거한 뒤 통계와 프롬프트를 만든다
- 글의 장르·구조는 `content_type`/`writing_instructions`(사용자 의도)가 최우선으로 결정한다. 사용자 의도는 outline 단계까지 전달된다(`OutlineAgentPort`/`OutlineAgentClient`/`WorkflowRunner`). 사진/OCR은 개요를 채우는 재료로만 쓴다
- 퀴즈/문제/힌트/정답/젤리/앱테크 OCR 기반 `퀴즈 정답 공유` 구조는 사용자 의도가 비어 있을 때만 적용되는 fallback이다. 오탐이 잦던 `포인트`는 트리거에서 제거함
- outline title이 비어도, 사용자 의도가 없을 때만 draft가 OCR 서비스명과 촬영일로 `[5월 15일] 모니스쿨 퀴즈 정답 공개｜오늘의 앱테크 퀴즈 정리` 같은 제목 fallback을 만든다. 사용자 의도가 있으면 이 fallback은 동작하지 않는다
- 검색 최적화는 (1) 콘솔의 `검색 키워드` 입력 → `targetKeywords`(영속, Flyway `V2__add_target_keywords.sql`)로 정형 전달, 또는 (2) `writing_instructions`/`content_type` 자연어의 "검색/SEO/키워드/노출/상위/최적화" 신호로 켜진다. "구글/티스토리"는 구글 SEO, 그 외는 네이버 SEO(기본값) 자동 분기. `target_keywords`가 있으면 그 자체로 사용자 의도로 간주(퀴즈 fallback off). 키워드 스터핑 금지
- `target_keywords`는 outline/draft 프롬프트에 주력 검색어로 주입되고, review_agent가 `seo_title_contains_keyword`/`seo_keyword_in_body`/`seo_no_keyword_stuffing`로 제목·본문 반영과 과다 반복(동일 검색어 4회+ 이고 본문 비중 >15%)을 issue로 검수한다. 키워드 미지정 시 SEO 검사 생략
- 문제 케이스 `u_74d258e5e27f4c60a78e86c9edb003b3`는 기존 사진 분석 결과를 재사용해 outline/draft/style/review 아티팩트를 다시 생성했다(사용자 의도 미전달 시절 사례)

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

- Flyway migration을 신규 schema 변경 때마다 추가하고, 운영 배포 전 `SPRING_JPA_HIBERNATE_DDL_AUTO=validate` 검증 유지

### 2. photo_grouping_agent

- 실제 사용자 샘플을 최소 5개, 전략은 3종 이상으로 추가해 `model_comparison_report` confidence를 높이기

### 3. 운영/UX 검증

- SSE 재연결/폴링 fallback의 브라우저 E2E 검증 추가
- 에이전트별 헬스 체크와 장애 메시지 표준화 — `adr/005-agent-error-format.md` 가 표준 안.
  단계 2a(orchestrator 인프라: `AgentInvocationException` / `@ControllerAdvice` / parser /
  TraceContextSanitizer / drift guard / coexistence test) **완료** →
  2b(voice_profile_agent reference) →
  2c(StyleAgentClient 정통 reference + `AgentHttpRetryer` 가 `retryable`/`retry_after_seconds`
  존중하도록 개선) →
  2d(`WorkflowController.executeRestyle` / `WorkflowRunner` 의 catch 가 raw 메시지 대신
  `getUserMessage()` 또는 sanitized fallback 을 `markFailed` 로 전달) →
  2e(Micrometer `agent.invocation.error` counter, tags=agent/error_code/status) →
  단계 3 mechanical migration 8개 →
  단계 4 legacy 제거 →
  단계 5(후속 ADR — circuit breaker / bulkhead) 순으로 진행한다

## 작업 시작 체크리스트

- Java 25 설치 확인
- ffmpeg 설치 확인
- Ollama 실행 여부 확인
- 필요한 모델(`qwen2.5vl:7b`, `qwen2.5:14b`, 필요 시 `gemma4`) 존재 확인
- `spring_orchestrator` 테스트 먼저 통과 확인
- `./scripts/verify-core.sh`로 Spring, 콘솔, 핵심 에이전트, 사진/동영상 파이프라인 영향 확인
- Docker 확인은 `./scripts/docker-up.sh` 또는 `./scripts/docker-up.sh momently-voice-profile momently-gateway` 실행 후 `http://127.0.0.1:18580`에서 확인
- 새 기능 추가 전 관련 테스트부터 작성

## 주의 사항

- 테스트가 실제로 통과하지 않은 상태를 완료로 간주하지 않는다.
- 스프링 쪽은 웹 DTO가 application 계층으로 새지 않도록 유지한다.
- 공개 API에는 인프라 설정값을 노출하지 않는다.
- 문서, ADR, 개발 일지는 코드 변경과 함께 갱신한다.
