# Roadmap

## 완료

- `photo_exif_llm_pipeline` 초기 파이프라인 구성
- 사진 정보 추출 / EXIF / bundle 생성
- `photo_grouping_agent` 생성
- 그룹화 전략 enum 도입
- 그룹화 FastAPI + Swagger UI 연결
- 공개 API와 내부 인프라 설정 경계 정리
- Spring 오케스트레이터 설계 문서 작성
- Spring 오케스트레이터 기본 구현
- 상태 머신 / 워크플로 러너 추가
- JPA persistence adapter 초안 추가
- Spring 테스트 커버리지 90% 이상 검증 추가

## 진행 중

- Spring 오케스트레이터 실제 외부 연동 구현
- 그룹화 규칙 정교화
- 전략별 점수 모델 분리
- `qwen2.5` / `gemma4` 비교 실험 준비

## 다음 우선순위

### 1. Spring 오케스트레이터 설계

- `PhotoInfoAgentClient`, `PhotoGroupingAgentClient` 실제 HTTP 호출 구현
- PostgreSQL 통합 테스트 및 스키마 전략 확정
- 실행 API 확장
- 실패/재시도/멱등성 시나리오 테스트 보강

### 2. photo_grouping_agent 고도화

- 전략별 규칙을 분리된 프로필 구조로 리팩터링
- LLM 보정 입력을 그룹 후보 요약 중심으로 축약
- gemma4 비교 실험
- 메타 부족 그룹 fallback 개선

### 3. hero_photo_agent 착수

- 그룹별 대표 사진 선택 기준 정의
- 입력/출력 스키마 설계
- API 명세 작성
- FastAPI 진입점 추가

## 이후 단계

### outline_agent

- 그룹 결과를 기반으로 문서 개요 생성

### draft_agent

- 개요를 바탕으로 초안 작성

### style_agent

- 원하는 문체/톤 반영

### review_agent

- 초안 검수 및 최종 정리

## 운영 정리 예정

- `shared/` 계층 도입 여부 결정
- Spring 설정 예시 정리
- 개발/운영 환경 분리
- 에이전트별 헬스 체크 및 모니터링 항목 정의
- 다른 PC/CI 환경에서도 동일하게 실행 가능한 Gradle wrapper 기준 정리
