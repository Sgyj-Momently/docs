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
- 사진/동영상 업로드 API와 콘솔 업로드 모드 추가
- 동영상 대표 프레임 추출 및 요약 병합 흐름 추가
- 동영상 프레임 샘플링 옵션을 Spring/Docker 설정까지 연결
- Docker compose 기준 사진/동영상 업로드 E2E 검증 완료
- 워크플로 실행/문체 재적용 상태 SSE 갱신 추가
- 실패 후 재실행 시 실패 메타데이터 정리 규칙 보강
- PostgreSQL 저장소 Testcontainers 통합 테스트 추가
- `photo_grouping_agent` LLM 보정 입력을 그룹 후보 중심으로 축약
- Docker compose healthcheck와 `service_healthy` 기동 조건 추가
- 워크플로 `retry` API와 콘솔 재시도 UX 추가
- 사용자-facing 실패 메시지에서 내부 컨테이너 경로 노출 제거
- 실패/재시도/멱등성 시나리오를 API 레벨 테스트로 보강
- 콘솔 결과물 서버 저장본/편집본 흐름 추가
- 작업 기록 검색/상태 필터와 상세 재시도 추적 추가
- 업로드 제한 조회 API와 콘솔 사전 검증 동기화
- 로그인 유지 옵션을 명시적으로 선택하는 방식으로 조정
- 에이전트 HTTP 호출 공통 connect/read timeout 설정 추가
- 주요 HTTP 에이전트 호출 공통 retry/backoff 정책 추가
- 결과물 수정본 latest 유지 + 버전 파일 보존 개수 제한 정책 추가
- `photo_grouping_agent` 전략별 의미 태그 프로필, 의미 점수 가중치, boundary evaluator 분리
- `photo_grouping_agent` 메타 부족 fallback에 파일명 숫자 순서 gap 분리 추가
- `photo_grouping_agent` 모델 비교 실행 스크립트와 CLI 저장 테스트 추가
- `photo_grouping_agent` LLM 보정 결과의 photo_id 커버리지 검증과 누락 ID 자동 복구 추가
- 실제 `qwen2.5:14b`/`gemma4:e4b` 비교 결과 생성. 현재 두 모델 모두 전체 커버/repair 없음
- 모델 비교 결과에 `quality_summary`를 추가해 커버리지, repair 수, 그룹 수 차이를 요약
- 모델 비교 결과에 `recommended_model`을 추가. 현재 두 샘플의 추천 모델이 갈리므로 추가 샘플 평가 필요
- 여러 비교 결과를 집계하는 `model_comparison_report` 스크립트 추가. 현재 2개 샘플 집계 기준 `gemma4:e4b` 추천
- 예제 입력 묶음 전체를 비교하고 리포트까지 갱신하는 `compare-sample-suite.sh` 추가
- 비교 CLI는 입력 JSON의 `grouping_strategy`를 기본값으로 사용하도록 수정. 현재 2개 샘플 suite 기준 `qwen2.5:14b` 추천
- suite 샘플 manifest(`examples/model_comparison_samples.json`)와 Ollama `temperature: 0` 비교 옵션 추가

## 진행 중

- 실제 사용자 샘플을 더 확보해 모델 비교 리포트 신뢰도 높이기

## 다음 우선순위

### 1. Spring 오케스트레이터 설계

- Testcontainers가 Docker Desktop 29 소켓을 안정적으로 잡도록 CI/로컬 실행 환경 정리
- 운영용 schema migration 전략 결정

### 2. photo_grouping_agent 고도화

- 추가 샘플 기반 그룹화 품질 튜닝

### 3. 운영 정리

- 에이전트별 헬스 체크 및 장애 메시지 표준화
- SSE 연결/재연결 동작을 브라우저 E2E로 검증
- 오래된 워크플로 원본 산출물 정리 정책 결정
- 다른 PC/CI 환경에서도 동일하게 실행 가능한 Docker/Testcontainers 설정 정리

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
