# Orchestrator Design

## 목적

Spring 오케스트레이터는 여러 에이전트 모듈을 정해진 순서대로 호출하고, 상태를 기록하며, 실패와 재시도를 관리하는 상위 워크플로 계층이다.

이 문서는 다음을 정의한다.

- 전체 파이프라인 순서
- 상태 머신
- 단계별 호출 조건
- 실패/재시도 정책
- Spring 설정 예시
- 저장 구조 예시

## 핵심 원칙

- 전체 파이프라인 순서는 Spring이 결정한다.
- FastAPI 에이전트는 자기 단계의 입력을 받아 결과만 반환한다.
- 각 단계 결과는 저장되어야 하며, 중간 실패 시 재실행 가능해야 한다.
- 공개 API는 도메인 의도만 전달하고, 내부 모델/주소 설정은 각 에이전트가 관리한다.
- 운영 메타데이터는 PostgreSQL에 저장하고, 큰 산출물은 artifact 저장소로 분리한다.
- 워크플로와 외부 공개 식별자의 기본 전략은 `UUIDv7`을 기준으로 한다.

## 전체 흐름

1. 사진 정보 추출 에이전트 호출
2. 사진 그룹화 에이전트 호출
3. 대표 사진 선택 에이전트 호출
4. 개요 생성 에이전트 호출
5. 초안 작성 에이전트 호출
6. 스타일 반영 에이전트 호출
7. 검수 에이전트 호출

## 상태 머신

### WorkflowStatus

- `CREATED`
- `PHOTO_INFO_EXTRACTING`
- `PHOTO_INFO_EXTRACTED`
- `PHOTO_GROUPING`
- `PHOTO_GROUPED`
- `HERO_PHOTO_SELECTING`
- `HERO_PHOTO_SELECTED`
- `OUTLINE_CREATING`
- `OUTLINE_CREATED`
- `DRAFT_CREATING`
- `DRAFT_CREATED`
- `STYLE_APPLYING`
- `STYLE_APPLIED`
- `REVIEWING`
- `REVIEW_COMPLETED`
- `COMPLETED`
- `FAILED`

## 단계별 입력/출력

### 1. photo_info_agent

- 입력:
  - `project_id`
  - 사진 폴더 또는 업로드 참조
- 출력:
  - 사진 정보 목록
  - 사진별 메타데이터
  - bundle JSON
- 성공 상태:
  - `PHOTO_INFO_EXTRACTED`

### 2. photo_grouping_agent

- 입력:
  - `project_id`
  - `grouping_strategy`
  - 사진 정보 목록
- 출력:
  - 그룹 목록
  - `group_reason`, `score`, `score_details`
- 성공 상태:
  - `PHOTO_GROUPED`

### 3. hero_photo_agent

- 입력:
  - `project_id`
  - 그룹 목록
  - 사진 정보 목록
- 출력:
  - 그룹별 대표 사진
- 성공 상태:
  - `HERO_PHOTO_SELECTED`

### 4. outline_agent

- 입력:
  - 그룹 목록
  - 대표 사진 정보
- 출력:
  - 문서 개요
- 성공 상태:
  - `OUTLINE_CREATED`

### 5. draft_agent

- 입력:
  - 개요
  - 그룹 정보
- 출력:
  - 초안
- 성공 상태:
  - `DRAFT_CREATED`

### 6. style_agent

- 입력:
  - 초안
  - 스타일 옵션
- 출력:
  - 스타일 반영 초안
- 성공 상태:
  - `STYLE_APPLIED`

### 7. review_agent

- 입력:
  - 스타일 반영 초안
  - 그룹/대표 사진 메타데이터
- 출력:
  - 최종 문서
- 성공 상태:
  - `REVIEW_COMPLETED`

## 단계 전이 규칙

- `PHOTO_GROUPING`은 `PHOTO_INFO_EXTRACTED` 이후에만 가능하다.
- `HERO_PHOTO_SELECTING`은 `PHOTO_GROUPED` 이후에만 가능하다.
- `OUTLINE_CREATING`은 `PHOTO_GROUPED`와 `HERO_PHOTO_SELECTED` 이후에만 가능하다.
- `DRAFT_CREATING`은 `OUTLINE_CREATED` 이후에만 가능하다.
- `STYLE_APPLYING`은 `DRAFT_CREATED` 이후에만 가능하다.
- `REVIEWING`은 `STYLE_APPLIED` 이후에만 가능하다.

## 실패 처리 정책

### 원칙

- 한 단계 실패가 전체 워크플로를 즉시 파괴하지는 않는다.
- 실패 상태와 실패 단계는 기록되어야 한다.
- 이미 성공한 이전 단계 산출물은 유지한다.
- 재실행 시 실패 단계부터 다시 시작할 수 있어야 한다.

### 권장 정책

- 네트워크 오류: 최대 3회 재시도
- 에이전트 4xx 응답: 재시도 없이 실패 처리
- 에이전트 5xx 응답: 제한적 재시도
- 타임아웃: 재시도 후 실패 처리
- 상태 기록:
  - `last_failed_step`
  - `last_error_code`
  - `last_error_message`

## 저장 구조 예시

### 저장소 전략

- 운영 메타데이터 저장소: `PostgreSQL`
- 대용량 산출물 저장소: 파일 시스템 또는 추후 `S3`/`MinIO` 계열
- 필요 시 보조 계층: `Redis`

### workflow 테이블

- `workflow_id`
- `project_id`
- `status`
- `grouping_strategy`
- `created_at`
- `updated_at`
- `last_failed_step`
- `last_error_message`

### workflow_step_result 테이블

- `workflow_id`
- `step_name`
- `status`
- `input_snapshot_path`
- `output_snapshot_path`
- `started_at`
- `finished_at`

### artifact 저장

- `artifacts/photo-info/bundle.json`
- `artifacts/photo-grouping/groups.json`
- `artifacts/hero-photo/hero.json`
- `artifacts/outline/outline.json`
- `artifacts/draft/draft.json`
- `artifacts/style/styled.json`
- `artifacts/review/final.json`

## Spring 설정 예시

```yaml
spring:
  datasource:
    url: jdbc:postgresql://127.0.0.1:5432/momently
    username: momently
    password: change-me

agents:
  photo-info:
    base-url: http://127.0.0.1:8100
  photo-grouping:
    base-url: http://127.0.0.1:8000
  hero-photo:
    base-url: http://127.0.0.1:8200
  outline:
    base-url: http://127.0.0.1:8300
  draft:
    base-url: http://127.0.0.1:8400
  style:
    base-url: http://127.0.0.1:8500
  review:
    base-url: http://127.0.0.1:8600
```

## Spring 코드 구조 예시

### 핵심 구성

- `WorkflowController`
- `WorkflowService`
- `WorkflowStateMachine`
- `AgentClientRegistry`
- `PhotoInfoAgentClient`
- `PhotoGroupingAgentClient`
- `HeroPhotoAgentClient`

### 호출 흐름 예시

1. `WorkflowController`가 작업 요청 수신
2. `WorkflowService`가 workflow 생성
3. 상태를 `PHOTO_INFO_EXTRACTING`으로 변경
4. `PhotoInfoAgentClient` 호출
5. 성공 시 결과 저장, 상태 변경
6. 다음 단계 호출

## 비동기 실행 권장

초기에는 단순 순차 실행으로도 가능하지만, 운영 관점에서는 비동기 작업 큐가 더 적합하다.

권장 초기 방향:

- API 요청은 workflow 생성 후 즉시 응답
- 실제 파이프라인은 백그라운드 작업으로 실행
- 상태 조회 API로 진행률 확인
- 각 단계는 멱등하게 설계하고, 재시도 시 중복 실행에 안전해야 한다

## 권장 API 예시

### 작업 생성

- `POST /api/v1/workflows`

### 작업 상태 조회

- `GET /api/v1/workflows/{workflowId}`

### 작업 재실행

- `POST /api/v1/workflows/{workflowId}/retry`

## 다음 작업

- Spring DTO 초안 작성
- 상태 enum을 코드 형태로 구체화
- `photo_grouping_agent` 호출용 AgentClient 예시 추가
- `hero_photo_agent` 계약 설계
