# Project Overview

## 목적

이 프로젝트는 여러 장의 사진을 입력받아 구조화된 정보로 변환하고, 목적에 맞게 그룹화한 뒤, 대표 사진 선택과 글 생성까지 이어지는 에이전트 기반 파이프라인을 만드는 것을 목표로 한다.

핵심 사용 사례는 다음과 같다.

- 여행 후기 작성
- 음식 후기 작성
- 사진 앨범 요약
- 이벤트 흐름 정리

## 전체 구조

현재 구조는 상위 워크스페이스 하나 아래에 에이전트별 모듈을 두는 방식이다.

```text
momently/
├── Agent.md
├── docs/
├── photo_exif_llm_pipeline/
└── photo_grouping_agent/
```

향후 구조는 아래 방향을 목표로 한다.

```text
momently/
├── shared/
├── orchestrator/
├── photo_info_agent/
├── photo_grouping_agent/
├── hero_photo_agent/
├── outline_agent/
├── draft_agent/
├── style_agent/
└── review_agent/
```

## 아키텍처 원칙

- 각 에이전트는 책임 단위로 독립 모듈로 유지한다.
- 전체 파이프라인 순서와 상태 관리는 Spring 오케스트레이터가 담당한다.
- FastAPI 에이전트는 자기 단계의 입력을 받아 결과를 반환하는 역할만 한다.
- Ollama와 모델 선택은 각 에이전트 내부 구현과 설정이 관리한다.
- 공개 API는 도메인 데이터와 사용자 의도만 받는다.

## 파이프라인 개요

1. 사진 정보 추출
2. 사진 그룹화
3. 대표 사진 선택
4. 개요 생성
5. 초안 작성
6. 스타일 반영
7. 검수

## 현재 구현 상태

### 1. 사진 정보 추출

- 모듈: `photo_exif_llm_pipeline`
- 역할:
  - 이미지 폴더 스캔
  - EXIF 추출
  - 사진별 요약 생성
  - bundle JSON 생성
- 특징:
  - Ollama 기반 `llava` 사진 분석
  - 텍스트 모델 기반 bundle 요약

### 2. 사진 그룹화

- 모듈: `photo_grouping_agent`
- 역할:
  - 사진 정보 목록을 그룹으로 묶기
  - 그룹화 전략 enum 적용
  - 규칙 기반 1차 그룹화
  - 선택적 LLM 보정
- 특징:
  - `TIME_BASED`
  - `LOCATION_BASED`
  - `SCENE_BASED`
  - `FOOD_TYPE_BASED`
  - `STORY_FLOW_BASED`
  - FastAPI `/docs` 지원

## 역할 분리

### Spring Orchestrator

- 에이전트 호출 순서 결정
- 작업 상태 관리
- 재시도 정책 관리
- 결과 저장
- 사용자 요청을 전략 enum으로 변환

### FastAPI Agents

- 입력 검증
- 단계별 핵심 로직 수행
- 내부 모델 호출
- 구조화 결과 반환

### Ollama

- 실제 로컬 모델 실행
- 에이전트 내부에서만 사용

## 공개 API와 내부 설정 구분

### 공개 API에 포함되는 것

- `project_id`
- `grouping_strategy`
- `time_window_minutes`
- `photos`

### 서버 내부 설정으로 관리하는 것

- `OLLAMA_BASE_URL`
- `OLLAMA_TIMEOUT_SECONDS`
- 전략별 기본 모델
- 내부 retry/fallback 정책

## 현재 핵심 문서

- 전역 규칙: [Agent.md](../Agent.md)
- 오케스트레이터 설계: [orchestrator-design.md](./orchestrator-design.md)
- 그룹화 API 문서: [photo_grouping_agent/docs/api-spec.md](../photo_grouping_agent/docs/api-spec.md)
- 그룹화 OpenAPI: [photo_grouping_agent/docs/openapi.yaml](../photo_grouping_agent/docs/openapi.yaml)
- 그룹화 오케스트레이션 메모: [photo_grouping_agent/docs/orchestration.md](../photo_grouping_agent/docs/orchestration.md)

## 다음 단계

- Spring 오케스트레이터 구조를 코드 수준으로 구체화
- 대표 사진 선택 에이전트 착수
- 전략별 그룹화 규칙 분리 강화
- gemma4 비교 실험
