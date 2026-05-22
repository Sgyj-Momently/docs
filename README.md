# Momently Docs

Momently 프로젝트의 공유 문서 저장소다. 아키텍처 결정, 모듈 간 계약, 설계 문서, 개발 규칙, 운영 설정을 한곳에서 관리한다.

---

## Momently 개요

Momently는 여행·음식·이벤트 등 사진·동영상을 입력받아 한국어 블로그 포스트를 자동 생성하는 에이전트 기반 파이프라인이다.

**파이프라인 흐름**

```
미디어 업로드
    → [photo_exif_llm_pipeline]  EXIF 추출 + LLM 시각 요약 + bundle JSON 생성
    → [photo_grouping_agent]     전략 기반 사진 그룹화 (FastAPI)
    → [spring_orchestrator]      워크플로 상태 관리 + 에이전트 순차 호출
    → [meta_agent]               메타 후보(제목·태그) 생성
    → [momently_console]         결과 확인·편집·네이버 발행 패키지 복사 (React)
```

모든 에이전트는 독립 모듈로 분리되어 있으며, Spring 오케스트레이터가 상태 머신 기반으로 전 단계를 조율한다. 각 FastAPI 에이전트는 자기 단계의 입·출력만 책임진다.

---

## 문서 인덱스

| 파일 | 설명 |
|------|------|
| [`Agent.md`](./Agent.md) | 워크스페이스 공통 개발·Git·커밋 규칙. 개별 프로젝트 `AGENT.md`의 상위 규칙 |
| [`project-overview.md`](./project-overview.md) | 전체 프로젝트 목적, 구조, 파이프라인 단계, 현재 구현 상태 요약 |
| [`contracts.md`](./contracts.md) | 모듈 간 데이터 계약(스키마·필드·아티팩트 경로·버전) 정의 |
| [`orchestrator-design.md`](./orchestrator-design.md) | Spring 오케스트레이터 파이프라인 순서, 상태 머신, 실패·재시도 정책 설계 |
| [`console-design.md`](./console-design.md) | Momently Console(React) UI 구조, 화면 흐름, API 연동 설계 |
| [`roadmap.md`](./roadmap.md) | 완료·진행 중·예정 기능 목록 및 우선순위 |
| [`dev-log.md`](./dev-log.md) | 날짜별 개발 작업 기록 |
| [`next-session-handoff.md`](./next-session-handoff.md) | 세션 인수인계 문서 — 현재 상태, 다음 작업 시작 순서, 주요 맥락 요약 |
| [`ai-team-harness.md`](./ai-team-harness.md) | AI 팀 운영 모델 — 파이프라인별 페어 구성, 하네스 운영 규칙 |
| [`CLOUDFLARE.md`](./CLOUDFLARE.md) | Cloudflare Tunnel(Zero Trust) 설정 — 호스트네임, nginx 게이트웨이 연동 방법 |
| [`adr/001-agent-module-boundary.md`](./adr/001-agent-module-boundary.md) | ADR 001: 에이전트 모듈 경계 결정 |
| [`adr/002-public-api-boundary.md`](./adr/002-public-api-boundary.md) | ADR 002: 공개 API 경계 결정 |
| [`adr/003-id-strategy.md`](./adr/003-id-strategy.md) | ADR 003: ID 전략 결정 (UUIDv7) |
| [`adr/004-database-strategy.md`](./adr/004-database-strategy.md) | ADR 004: 데이터베이스 전략 결정 (PostgreSQL) |
