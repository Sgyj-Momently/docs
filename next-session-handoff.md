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

- 폴더 단위 이미지 스캔 가능
- EXIF 추출 가능
- Ollama `llava` 기반 이미지 요약 가능
- bundle JSON 생성 가능
- writer model과 vision model 분리 가능
- `POST /api/v1/photo-info` FastAPI 진입점 추가됨
- `project_id`를 받아 내부 설정의 input/output root 기준으로 bundle 생성 가능

### photo_grouping_agent

- 그룹화 전략 enum 기반 입력 계약 존재
- FastAPI 진입점 존재
- `/docs`, `/openapi.json` 확인 가능
- 공개 API에서 `ollama_base_url`, `ollama_timeout_seconds` 제거 완료
- `POST /api/v1/photo-groups`는 `photos` 직접 입력과 `photo_info_bundle_path` 기반 입력을 모두 지원
- `photo_info_bundle_path` 입력 시 bundle JSON을 읽어 grouping `photos` 입력으로 변환

### spring_orchestrator

- 헥사고날 구조 초안 존재
- `WorkflowController`, `WorkflowService`, `WorkflowStateMachine`, `WorkflowRunner` 구현됨
- `POST /api/v1/workflows/{workflowId}/run` 실행 API 추가됨
- `PhotoInfoAgentClient`는 RestClient 기반 HTTP 호출로 교체됨
- `PhotoGroupingAgentClient`는 RestClient 기반 HTTP 호출 구조 존재
- `WorkflowRunner`는 photo-info 결과의 `photoCount`, `photoInfoBundlePath`를 기록하고 photo-grouping payload로 전달함
- `memory` 프로필 저장소와 `postgres` 프로필 JPA adapter 존재
- 테스트 및 JaCoCo 커버리지 검증 통과

## 스프링 표준 검증 명령

현재 환경 차이를 줄이기 위해 아래 명령을 표준으로 사용한다.

```bash
cd spring_orchestrator
env GRADLE_USER_HOME=.gradle-home GRADLE_OPTS='-Dorg.gradle.native=false' gradle test jacocoTestReport jacocoTestCoverageVerification
```

## 현재 기준 의사결정

- 공개 워크플로 식별자 기본 전략: `UUIDv7`
- 운영 메타데이터 DB 기본 전략: `PostgreSQL`
- 대용량 산출물은 DB 본문이 아니라 artifact 저장소에 분리
- 전체 파이프라인 순서와 상태 머신은 Spring 오케스트레이터가 관리
- 에이전트는 자기 단계 입력을 받아 결과만 반환

## 바로 다음 우선 작업

### 1. spring_orchestrator

- 실제 배치/스토리지 환경에 맞게 `project_id -> input/output 경로` 매핑 방식 확정
- 단계별 artifact 경로를 별도 step result 저장소로 분리할지 결정
- PostgreSQL 통합 테스트 추가
- 실패/재시도/멱등성 시나리오 테스트 보강

### 2. photo_grouping_agent

- LLM 보정 입력 축약
- 전략별 규칙 분리 리팩터링
- `gemma4` 비교 실험

### 3. 다음 에이전트

- `hero_photo_agent` 모듈 생성
- 입력/출력 스키마와 API 문서 작성

## 작업 시작 체크리스트

- Java 25 설치 확인
- Ollama 실행 여부 확인
- 필요한 모델(`llava`, `qwen2.5:14b`, 필요 시 `gemma4`) 존재 확인
- `spring_orchestrator` 테스트 먼저 통과 확인
- 새 기능 추가 전 관련 테스트부터 작성

## 주의 사항

- 테스트가 실제로 통과하지 않은 상태를 완료로 간주하지 않는다.
- 스프링 쪽은 웹 DTO가 application 계층으로 새지 않도록 유지한다.
- 공개 API에는 인프라 설정값을 노출하지 않는다.
- 문서, ADR, 개발 일지는 코드 변경과 함께 갱신한다.
