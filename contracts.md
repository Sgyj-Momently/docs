# 모듈 간 계약 & 아티팩트 규약

이 문서는 `photo_exif_llm_pipeline` → `photo_grouping_agent` → `spring_orchestrator`가
서로 데이터를 전달할 때 지켜야 하는 최소 계약(스키마/필드/경로/버전)을 정의한다.

## 1) 용어

- **Contract**: 모듈 간 경계에서 합의된 입력/출력 스키마(필드명, 타입, 의미).
- **Artifact**: 대용량 산출물(JSON/Markdown)을 파일로 저장한 결과물. DB에는 “경로/요약”만 저장하는 것을 기본으로 한다.
- **Public Output**: 사용자에게 노출 가능한 결과물(블로그, 그룹화 입력 등). 민감/제외 대상은 포함되면 안 된다.

## 2) 원칙

- **공개 API는 도메인 의도만** 받는다. 인프라 설정(주소/timeout/모델명)은 요청 계약에 포함하지 않는다.
- **대용량 본문은 artifact로** 남긴다. (오케스트레이터는 경로와 요약만 들고 간다.)
- **하위호환을 우선**한다.
  - 새 필드는 기본적으로 optional/nullable로 추가한다.
  - 필수 필드를 추가해야 하면 “버전 업” 또는 “새 엔드포인트/새 artifact”로 분리한다.
- **민감정보 제외 정책은 경계를 넘지 않는다.**
  - `exclude_from_public_outputs=true`인 사진의 OCR/요약/위치 힌트는 다음 단계로 전달되지 않아야 한다.

## 3) 단계별 계약(요약)

### A. Photo Info 산출물 (bundle)

- **대표 artifact**: `bundle.json`
- **최소 요구 필드(오케스트레이터 관점)**:
  - `photo_count` (int, >=0)
  - `photos` (array)
  - 각 `photo`는 아래 필드들을 가질 수 있다(선택적):
    - `file_name`
    - `captured_at`
    - `has_gps`
    - `gps`
    - `photo_summary` (object)
      - `exclude_from_public_outputs` (boolean)
      - `summary`, `subjects`, `scene_type`, `location_hint` 등(선택)

> 오케스트레이터는 bundle 본문을 DB에 저장하지 않는다. 경로 + 요약만 유지한다.

### B. Photo Grouping 요청/응답

- **요청(오케스트레이터 → 그룹화 에이전트) 최소 필드**
  - `project_id` (string)
  - `grouping_strategy` (string enum)
  - `photos` (array)
    - `photo_id` (string)
    - `file_name` (string)
    - `captured_at` (string | null)
    - `has_gps` (boolean | null)
    - `gps` (object | null)
    - `location_hint` (string | null)
    - `scene_type` (string | null)
    - `summary` (string | null)
    - `subjects` (array)

- **응답(그룹화 에이전트 → 오케스트레이터) 최소 필드**
  - `grouping_strategy` (string enum)
  - `group_count` (int)
  - `groups` (array, optional; 클 수 있으므로 요약만 읽고 전체는 artifact로 저장 가능)

### C. Hero Photo(대표 사진 선택) 요청/응답 (초안)

- **입력(오케스트레이터 → hero_photo_agent) 최소 필드**
  - `project_id` (string)
  - `groups` (array)
    - `group_id` (string)
    - `photo_ids` (string[])
  - `photos` (array)
    - `photo_id` (string)
    - `file_name` (string)
    - `summary` (string | null)
    - `ocr_text` (string[])
    - `confidence` (number | null)

- **출력(hero_photo_agent → 오케스트레이터) 최소 필드**
  - `hero_photo_count` (int)
  - `hero_photos` (array)
    - `group_id` (string)
    - `hero_photo_id` (string)
    - `reason` (string)

> 초기 품질 정책은 `confidence` 기반 선택을 우선하고, 동률일 때는 summary/ocr_text의 “풍부함”을 보조 기준으로 사용한다.

### photo_id 규칙(안정성)

- 그룹화(B)와 대표 사진(C)은 **같은 photo_id**를 공유해야 downstream이 안정적으로 연결된다.
- 오케스트레이터가 그룹화 요청 payload를 만들 때 photo_id는 **파일명 기반**으로 안정화한다.
  - 예: `photo_id = "file:" + file_name`

### D. Outline(개요 생성) 요청/응답 (초안)

- **입력(오케스트레이터 → outline_agent) 최소 필드**
  - `project_id` (string)
  - `groups` (array)
    - `group_id` (string)
    - `photo_ids` (string[])
    - `location_hint` (string | null)
    - `group_reason` (string | null)
  - `hero_photos` (array)
    - `group_id` (string)
    - `hero_photo_id` (string)
    - `reason` (string)
  - `photos` (array)
    - `photo_id` (string)
    - `file_name` (string)
    - `summary` (string | null)
    - `ocr_text` (string[])
    - `confidence` (number | null)

- **출력(outline_agent → 오케스트레이터) 최소 필드**
  - `outline_status` (string: ok | error:...)
  - `outline` (object | null)
    - `title` (string | null)
    - `sections` (array)
      - `section_id` (string)
      - `heading` (string)
      - `bullets` (string[])
      - `supporting_photo_ids` (string[])
    - `tone` (string | null)

## 4) 아티팩트 경로 규약(권장)

오케스트레이터가 단계 결과를 저장할 때 “예측 가능한 경로”를 유지한다.

- Photo Info
  - `.../<projectId>/bundles/bundle.json`
  - `.../<projectId>/blog.md` (선택)
- Photo Grouping
  - `.../<projectId>/grouping/grouping-result.json`
- Hero Photo
  - `.../<projectId>/hero-photo/hero-result.json`
- Outline
  - `.../<projectId>/outline/outline.json`

> 추후 7단계 확장 시에도 동일하게 `.../<projectId>/<step>/...` 형태로 확장한다.

## 5) 버전/변경 절차

### 계약 변경이 아닌 경우(자유 변경)

- 내부 리팩터링
- 로그/메트릭/성능 개선
- 테스트 보강
- optional 필드의 추가(기존 소비자가 무시 가능)

### 계약 변경인 경우(절차 필수)

아래 중 하나라도 해당하면 계약 변경으로 취급한다.

- 필드명 변경/삭제
- 필드 타입 변경
- 필수 필드 추가
- 의미 변경(같은 필드인데 해석이 바뀜)

절차:

1. 이 문서(`docs/contracts.md`)에 변경 내용을 먼저 반영한다.
2. 생산자(Producer) 모듈부터 변경한다.
3. 소비자(Consumer) 모듈이 구버전/신버전을 동시에 처리할 수 있게 이행 기간을 둔다.
4. 구버전 제거는 별도 PR로 수행한다.

## 6) 민감정보/제외 정책 체크

- Photo Info 단계에서 `exclude_from_public_outputs=true`가 붙은 사진은
  - 그룹화 요청 payload에서 제외하거나,
  - 최소 메타데이터만 전달하고 텍스트/위치 힌트는 마스킹해야 한다.
- “블로그/공개 문서”에는 제외 사진이 절대 포함되면 안 된다.

## 7) 관련 문서

- 전체 설계/상태 머신: `./orchestrator-design.md`
- 팀 운영/하네스: `./ai-team-harness.md`

