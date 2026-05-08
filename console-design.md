# Momently Console — UI 설계

## 개요

`momently_console`은 Momently 파이프라인을 조작하고 결과를 확인하는 React 기반 웹 콘솔이다.  
Vite + React 19 + React Router + lucide-react로 구성된다.

## 실행

```bash
cd momently_console
npm run dev   # http://localhost:5173
```

API Base URL은 `.env`로 관리한다.

```
# .env
VITE_API_BASE_URL=http://127.0.0.1:18080
```

## 페이지 구성

### 1. 새 글 쓰기 (Write)

파이프라인을 통해 블로그 글을 생성하는 메인 기능 페이지.

**입력 항목**

| 항목 | 설명 |
|------|------|
| 미디어 | 사진/동영상 직접 업로드 (드래그 앤 드롭 지원). 서버 묶음 ID 입력은 고급 옵션 |
| 콘텐츠 유형 | 블로그 / 여행후기 / 음식후기 / 체험단 / 이벤트 |
| 체험단 규칙 | 최소 사진 수, 간판 노출 여부, 상호명 언급 여부, 자유 입력 규칙 |
| 작성 방향 | 사용자가 원하는 내용 방향 자유 입력 |
| 말투 선택 | 프리셋 또는 사용자 저장 말투 적용 |
| 고급 옵션 | 그룹화 전략, 시간 윈도우 (기본값으로 충분) |

**실행 흐름**
1. 서버 업로드 정책 조회 (`GET /api/v1/uploads/config`)
2. 업로드 모드에서는 미디어 저장 (`POST /api/v1/uploads/media`) 후 새 프로젝트 ID 수신
3. 워크플로 생성 (`POST /api/v1/workflows`)
4. 즉시 실행 (`POST /api/v1/workflows/{id}/run`)
5. SSE 구독 (`GET /api/v1/workflows/{id}/events`)으로 상태 갱신
   - SSE 연결 실패 시 2초 폴링으로 fallback
6. `COMPLETED` 도달 시 아티팩트 탭 표시

> **참고**: 기본 흐름은 업로드 모드다. 서버 묶음 ID 입력은 이미 서버 입력 폴더에 준비된 미디어를
> 재사용하거나 디버깅할 때만 고급 옵션에서 사용한다.
> 콘솔은 서버가 내려준 업로드 개수/용량/확장자 제한을 기준으로 사전 검증하며, 중복 선택한 파일은 제외한다.

### 2. 말투 설정 (Tone)

글쓰기에 적용할 문체/말투를 관리하는 페이지.

**기본 프리셋 (수정 불가)**

| ID | 이름 | 설명 |
|----|------|------|
| `mz` | MZ 말투 | 친근하고 캐주얼한 MZ세대 스타일 |
| `formal` | 정중한 말투 | 격식 있고 품격 있는 문체 |
| `emotional` | 감성 글 | 서정적이고 감성적인 문체 |
| `info` | 정보형 | 사실 위주의 깔끔한 정보 전달 |

**사용자 말투**
- `voice_profile_agent`에 프로필과 샘플을 저장한다.
- TipTap 에디터에 기존 글과 이미지를 붙여넣을 수 있으며, 이미지는 학습 저장 전에 JPEG data URL로 축소한다.
- 새 글 쓰기 페이지의 말투 선택에서 서버 프로필을 프리셋처럼 사용 가능하다.

### 3. 작업 기록 (History)

과거 워크플로 목록 조회 페이지.

- `GET /api/v1/workflows`로 서버 저장소(Postgres/메모리 프로필)의 워크플로 목록 조회
- 각 항목: `workflowId`, `projectId`, `groupingStrategy`, `status`
- 검색/상태 필터로 기록을 좁힐 수 있다.
- 클릭 시 API에서 최신 상태 조회 + 아티팩트 표시
- 실패한 작업은 상세 화면에서 바로 `retry` 가능하며, 재시도 후 SSE/폴링으로 진행 상태를 추적한다.
- `DELETE /api/v1/workflows`로 워크플로 메타데이터 기록 전체 삭제

### 문체 다시 적용

- 완료된 워크플로에서 `POST /api/v1/workflows/{id}/restyle` 호출
- 서버는 `STYLE_APPLYING → REVIEWING → COMPLETED` 상태 이벤트를 SSE로 발행
- 콘솔은 SSE를 우선 사용하고, 실패하면 기존 결과 아티팩트 폴링으로 fallback
- 최종 결과물은 화면에서 편집 가능하며, `POST /api/v1/workflows/{id}/artifacts/{type}/edits`로 서버 저장본을 남긴다.

### 4. 파이프라인 모니터 (Monitor)

개발/디버그용 로우레벨 콘솔.

- Project ID 직접 입력 후 Create / Run
- 파이프라인 스텝 + 메트릭 실시간 확인
- 모든 아티팩트 탭 접근 가능

## 파이프라인 스텝

```
준비 → 사진 분석 → 민감정보 → 품질 → 그룹화 → 대표 사진 → 개요 → 초안 → 문체 → 검수 → 완료
```

상태값 매핑:

| Status | 단계 |
|--------|------|
| `CREATED` | 준비 |
| `PHOTO_INFO_EXTRACTING` / `PHOTO_INFO_EXTRACTED` | 사진 분석 |
| `PRIVACY_REVIEWING` / `PRIVACY_REVIEWED` | 민감정보 |
| `QUALITY_SCORING` / `QUALITY_SCORED` | 품질 |
| `PHOTO_GROUPING` / `PHOTO_GROUPED` | 그룹화 |
| `HERO_PHOTO_SELECTING` / `HERO_PHOTO_SELECTED` | 대표 사진 |
| `OUTLINE_CREATING` / `OUTLINE_CREATED` | 개요 |
| `DRAFT_CREATING` / `DRAFT_CREATED` | 초안 |
| `STYLE_APPLYING` / `STYLE_APPLIED` | 문체 |
| `REVIEWING` / `REVIEW_COMPLETED` | 검수 |
| `COMPLETED` | 완료 |

## 아티팩트 타입

| 키 | 내용 |
|----|------|
| `blog` | 최종 블로그 글 (markdown) |
| `review` | 검수 결과 |
| `draft` | 초안 |
| `outline` | 개요 |
| `bundle` | 사진 정보 번들 |
| `quality` | 품질 점수 |
| `grouping` | 그룹화 결과 |
| `hero` | 대표 사진 선택 결과 |
| `style` | 문체 적용 결과 |

## 기술 구성

| 항목 | 내용 |
|------|------|
| 프레임워크 | React 19 |
| 빌드 도구 | Vite 7 |
| 아이콘 | lucide-react |
| 상태 | React useState (로컬) + localStorage (영속) |
| 라우팅 | react-router-dom |
| 스타일 | 단일 CSS 파일 + CSS variables |
| 환경변수 | `VITE_API_BASE_URL` |

## 개선 예정

- 마크다운 렌더러 교체 (`marked` 등)
- 에러/성공 토스트 알림 분리
- 모바일 최적화 (하단 네비게이션)
