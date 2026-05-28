# ADR 007 - Naver Publisher Browser Extension

## 상태

초안 (Proposed)

## 날짜

2026-05-28

## 맥락

Momently 워크플로는 사진 → 개요 → 초안 → 문체 → 검수 → 메타(네이버 SEO) 단계를
거쳐 최종 한국어 블로그 마크다운 + 발행 메타데이터(제목 후보 / 해시태그 /
메타 디스크립션)를 생성한다. 현재 사용자가 결과를 실제 네이버 블로그에 올리는
경로는 `momently_console` 의 **발행 패키지 클립보드 복사**(`publicationPackage.js`)
뿐이다.

**현재 방식의 한계 (실측):**

1. **마크다운 미지원**: 네이버 SmartEditor 3.0 은 마크다운 파서가 없다. 클립보드의
   `## 제목` / `**굵게**` / `![](IMG_0001.jpg)` 가 평문으로 들어가거나 무시된다.
2. **이미지 전달 불가**: `draft_agent` 가 본문에 삽입하는 이미지는
   `![alt](IMG_0001.jpg)` 처럼 **파일명만** 가진다 (외부 URL 도 아님). 붙여넣어도
   네이버가 로드할 위치가 없어 이미지가 통째로 누락된다. 사용자가 사진을 에디터에서
   하나씩 다시 업로드해야 한다.
3. **수작업 과다**: 제목·해시태그·본문·이미지를 사용자가 각각 옮겨야 한다.

**자동 발행이 막혀 있는 이유:**

- 네이버 공식 블로그 글쓰기 OpenAPI 는 신규 발급이 사실상 중단됨 (조회 일부만 잔존).
- 서버(orchestrator)에서 사용자 세션으로 자동 발행하는 것은 네이버 이용약관 §7
  (자동화 수단 접근 금지) 위반이며, 다수 계정 대량 발행으로 간주되면 정보통신망법
  책임까지 발생할 수 있다.

**남은 현실적 경로:**

사용자 **본인 PC 에서, 본인 네이버 세션으로, 본인이 발행 버튼을 직접 클릭**하는
브라우저 확장. 본인이 자기 계정 작성을 보조받는 형태라 약관 위반 소지가 가장 낮고,
SmartEditor 의 자체 이미지 업로드 경로를 그대로 태워 네이버 CDN 에 정상 저장된다.

## 결정

**`momently-publisher` — Chrome / Whale (Chromium, manifest v3) 브라우저 확장 신규 레포** 도입.

### 컴포넌트 & 데이터 흐름

```
momently_console (결과 화면)
   │  ① 사용자가 "네이버로 보내기" 클릭
   │  ② window.postMessage({type, payload}) — payload = {
   │         title, hashtags, metaDescription,
   │         blocks: [ {kind:"text", markdown}, {kind:"image", url, alt} ... ],
   │         imageUrls: [signed URL ...]   // 짧은 TTL signed URL
   │     }
   ▼
content script (momently_console 도메인에서만 주입)
   │  ③ origin 검증 후 background 로 chrome.runtime.sendMessage
   ▼
background service worker
   │  ④ signed URL 로 이미지 binary fetch (CORS 허용된 storage)
   │  ⑤ 네이버 블로그 글쓰기 탭으로 메시지 relay
   ▼
content script (blog.naver.com / SmartEditor iframe)
   ⑥ 본문 블록 순서대로 contenteditable 에 주입
   ⑦ 이미지 blocks 는 SmartEditor 의 파일 업로드 input(DataTransfer) 에 binary 주입
      → 네이버가 자기 CDN 에 업로드 → 본문에 정상 이미지로 배치
   ⑧ 제목/해시태그 필드 자동 채움
   ⑨ ★ 발행 버튼은 채우기만 하고 클릭하지 않음 — 사용자가 직접 검토 후 발행
```

### 핵심 결정 사항

| # | 결정 | 근거 |
|---|---|---|
| 1 | **콘솔 → 확장 postMessage 핸드셰이크** | 콘솔이 이미 로그인/워크플로 세션 보유 — 확장이 별도 인증/토큰 관리 불필요. 확장은 콘솔이 건넨 payload 만 처리 |
| 2 | **이미지는 signed URL → 확장이 binary fetch → SmartEditor file input 주입** | 네이버 자체 CDN 업로드 경로를 태워 외부 호스팅 의존 0, 사용자 블로그에 정상 표시. postMessage 로 binary 직접 전달하면 payload 가 비대 → URL 만 전달 |
| 3 | **발행 버튼 자동 클릭 금지 — 필드 자동 채움까지만** | 약관 회색지대의 안전선. "본인이 발행 버튼을 직접 누른다" 가 자동화 발행과 본인 작성 보조를 가르는 경계 |
| 4 | **manifest v3, Chrome + Whale 타겟** | 둘 다 Chromium 기반 코드 공유. Whale 은 네이버 사용자층 + 네이버 로그인 세션 상존 비율 높음 |
| 5 | **content script 주입 도메인 화이트리스트** | `momently_console` 오리진 + `*.blog.naver.com` 두 곳에만 주입. 그 외 페이지 접근 권한 없음 |
| 6 | **remote config kill switch** | 확장이 기동 시 우리 서버의 config (예: `GET /api/v1/extension/config`) 를 조회해 `enabled:false` 면 즉시 기능 비활성화. 네이버 정책 변경·차단 탐지 시 스토어 심사(수일) 없이 원격으로 끌 수 있는 안전장치 |

### postMessage 보안

origin 검증만으로는 부족하다 (콘솔 origin 에 XSS 가 있으면 동일 origin 메시지로 우회).
다층 방어를 둔다.

- **target origin 명시**: 콘솔 측 `window.postMessage(payload, EXTENSION_BRIDGE_ORIGIN)` 에서
  `EXTENSION_BRIDGE_ORIGIN` 은 와일드카드 금지, 구체값 (예: `https://console.momently.app`).
  content script 측은 `event.origin === EXTENSION_BRIDGE_ORIGIN` 일 때만 수신
- **one-time nonce + replay 방어**: payload 에 콘솔이 발급한 nonce 포함. background 가
  처리한 nonce 를 기억(짧은 LRU)해 중복 nonce 는 거부. 캡처-재전송(replay) 차단
- **payload 스키마 검증**: 허용 origin 이라도 malformed payload (필수 필드 누락, 과대 크기,
  비정상 값) 는 graceful reject. title/markdown 은 길이 cap + 제어문자 sanitize
- **imageUrls SSRF 방지**: signed URL 의 host 를 storage 도메인 allowlist 로 제한.
  내부 네트워크 주소(`169.254.169.254`, `localhost`, `10./192.168./172.16.` 사설망, `file:`,
  `chrome-extension:` 등) 는 거부. background 의 fetch 는 이 allowlist 통과 URL 만 수행
- **위협 경계 선언**: 콘솔 origin 자체의 XSS 는 이 확장의 보안 경계 밖이며, 콘솔의 CSP +
  XSS 방어가 1차 책임. 확장은 nonce/스키마/SSRF 검증으로 2차 방어
- **세션 불간섭**: 확장은 네이버 세션 쿠키를 읽거나 저장하지 않는다 — DOM 주입만 수행

### 이미지 전달 디테일 (구현 시 검증 필요)

- **signed URL TTL ↔ 이미지 수량 경쟁 조건**: 5분 TTL 로 시작하되, background 의 binary
  fetch 는 **순차가 아닌 병렬**(`Promise.all`) 로 실행해 TTL 내 완료를 노린다. 그래도
  만료되면 background 가 콘솔에 재발급을 요청하는 메시지 프로토콜로 fallback (확장의
  stateless 원칙 유지 — 토큰을 들지 않고 콘솔에 위임). 이미지 상한을 정해 TTL 을 산정
  (예: 최대 30장 기준 5분)
- **확장 ID ↔ storage CORS**: manifest v3 의 background fetch origin 은
  `chrome-extension://<id>` 이며 dev(unpacked) ↔ 스토어 배포 시 ID 가 달라진다. 또한
  다수 object storage(S3/R2/MinIO) 는 `chrome-extension:` scheme 을 CORS origin 으로
  인식하지 않는다. 대응: (1) `manifest.json` 에 `key` 를 고정해 ID 안정화, (2) signed URL
  자체에 CORS 응답 헤더를 포함하거나, fetch 를 `no-cors` + opaque response → blob 변환
  경로로 우회. 두 방식의 실현성을 7c 에서 PoC 로 확정
- **SmartEditor file input 주입 (PoC 선행 필수)**: SmartEditor 3.0 의 이미지 업로드가
  `<input type="file">` 인지 custom drag-drop handler 인지, 리스너가 `change`/`drop`/`input`
  중 무엇에 바인딩됐는지에 따라 주입 방식이 달라진다. Chromium 은 `input.files =
  dataTransfer.files` 를 보안상 제한하는 버전이 있다. 따라서 7c 초입에 **DOM 구조 탐사 +
  DataTransfer 주입 PoC** 를 선행하고, 실패 시 fallback(드래그 이벤트 시뮬레이션 / paste
  event 시뮬레이션) 을 둔다. 이 주입은 PoC 미완 상태이며 7c 에서 실현성을 확정한다

### 단계별 적용

| 단계 | 산출물 | 검증 기준 |
|---|---|---|
| **7a** | 신규 레포 `momently-publisher` scaffold (manifest v3, content/background 골격). 콘솔 ↔ 확장 postMessage 핸드셰이크 + origin/nonce 검증. payload echo 만 (네이버 주입 없음) | (1) 콘솔에서 "네이버로 보내기" → 확장이 payload 수신 로그. (2) 타 origin 메시지 무시. (3) **malformed payload (필드 누락/과대 크기/비정상 scheme) graceful reject**. (4) **imageUrls 에 내부망 주소(`169.254.169.254`/`localhost`/사설망) 포함 시 SSRF 거부**. (5) 중복 nonce 거부 |
| **7b** | SmartEditor content script — 제목/본문 텍스트 블록 주입 (이미지 제외). 발행 버튼 미클릭. **SmartEditor 셀렉터 단일 모듈화 + 미스매치 시 graceful 실패(사용자에게 "확장 업데이트 필요" 안내)** | 실제 blog.naver.com 글쓰기에서 제목+본문 텍스트가 채워짐. 발행은 수동. 셀렉터 미스매치 시 조용히 깨지지 않고 사용자 안내 |
| **7c** | 이미지 파이프라인. **선행: SmartEditor file input 메커니즘 PoC (DataTransfer 주입 성공 여부 + 실패 시 fallback 확정)**. signed URL 병렬 fetch(+ 만료 재발급) + file input 주입. 본문 내 위치 보존. storage CORS ↔ 확장 ID 처리 확정 | (1) PoC 로 주입 방식 확정. (2) 사진 N장이 네이버 CDN 에 업로드되고 본문 올바른 위치에 배치. (3) TTL 만료 재발급 fallback 동작 |
| **7d** | 콘솔 연동 마감 — `publicationPackage.js` 를 확장 payload 빌더로 확장(blocks 구조 생성). 기존 클립보드 복사 fallback 유지 | 확장 미설치 사용자는 클립보드 복사, 설치 사용자는 1-클릭(검토 후 발행) |
| **7e** (후속) | 웹스토어 심사 제출 (Chrome Web Store + Whale 스토어). 권한 최소화 정당화 문서. **네이버 정책 모니터링 책임자/주기 지정 + (가능 시) 네이버 사전 문의/파트너 확인 + 법무 검토 후 사용자 ToS 고지 문구** | 스토어 승인 + 모니터링 owner 지정 + kill switch endpoint 운영 |

7a-d 가 MVP. 7e 는 배포 채널 결정 후.

### orchestrator / 콘솔 측 변경

- `publicationPackage.js` 를 텍스트 평면 조립 → **blocks 구조 빌더**로 확장:
  본문 마크다운을 text/image 블록 시퀀스로 파싱 (이미지 위치 보존)
- 이미지 signed URL 발급: orchestrator 가 이미지 storage 의 short-TTL signed URL 을
  워크플로 결과 응답에 포함하거나 별도 endpoint 제공 (storage CORS 에 확장 background fetch 허용)
- WorkflowStatus 변경 **없음** — 발행은 워크플로 바깥(사용자 수동)이라 도메인 상태에 단계 추가 안 함

## 이유

**왜 브라우저 확장인가 (서버 자동화 대신)?**
- 서버 자동화는 약관 §7 위반 + 봇 탐지(계정 정지) + 2FA/캡차 우회 부담 + 형사 책임 가능성
- 확장은 사용자 본인 세션·본인 클릭이라 "본인 작성 보조" 로 해석돼 회색지대 위험이 가장 낮음
- 네이버 세션·쿠키를 우리 서버가 만지지 않으므로 자격증명 보관 책임 0

**왜 발행 버튼을 자동 클릭하지 않는가?**
- "필드 자동 채움" 과 "자동 발행" 사이가 약관 해석의 경계. 사용자가 최종 검토 후
  직접 발행하면 자동화 게시가 아니라 작성 편의 도구로 남는다
- 동시에 AI 생성 글이 무검토로 대량 발행되는 품질·신뢰 리스크도 차단

**왜 이미지를 binary 주입하는가 (public URL 대신)?**
- 네이버 SmartEditor 는 외부 `<img src>` 를 자기 CDN 으로 자동 복사하지 않는다 →
  public URL 방식은 외부 호스팅 의존 + 사용자 블로그에서 외부 이미지가 깨질 위험
- file input 에 binary 를 주입하면 네이버 정식 업로드 경로를 타 자기 CDN 저장 → 영구

**왜 postMessage (확장 자체 로그인 대신)?**
- 콘솔이 이미 인증·워크플로 컨텍스트를 가짐. 확장이 토큰을 또 관리하면 자격증명
  표면적이 늘고 동기화 버그 위험. 확장은 stateless 주입기로 유지

## 결과

**Pros**
- 사용자 수작업(제목·본문·사진 N장 재업로드)이 1-클릭(+ 검토 후 발행)으로 축소
- 이미지가 네이버 CDN 에 정상 저장 — 현재 방식의 최대 결함 해소
- 서버 자동화의 법적·계정 리스크 회피
- 확장이 stateless 라 보안 표면적 최소

**Cons / 위험**
- **네이버 UI 의존성**: SmartEditor DOM/셀렉터가 바뀌면 content script 깨짐 → 유지보수 부담. 셀렉터를 한 곳에 모으고 버전 가드 필요
- **약관 회색지대 (정면 인정)**: 네이버 약관 §7 의 "자동화 수단" 해석에 **content script 의
  프로그래매틱 DOM 자동 채움 자체가 포함될 가능성**이 있다 — 발행 버튼 비클릭만으로
  완전 면책되지 않는다. 대응:
  - (1) 결정 6 의 **remote config kill switch** — 정책 변경·차단 탐지 시 원격으로 즉시 비활성화
  - (2) **네이버 정책 모니터링 책임자/주기 지정** (운영 항목, 단계 7e 에 포함) — 약관 개정·SmartEditor
    변경 추적
  - (3) 웹스토어 심사(7e) 전 **네이버 측 사전 문의 / 파트너 채널 확인** 검토 — 가능하면 명시적 허용 획득
  - (4) 법무 검토 후 ToS 위험을 사용자에게 고지 (확장 설치 시 동의 문구)
- **신규 레포 + 웹스토어 심사**: 배포·심사·업데이트 채널이 추가됨. manifest v3 권한(activeTab, scripting, host_permissions)을 최소로 정당화해야 심사 통과
- **postMessage 보안**: origin 검증 누락 시 임의 페이지가 확장에 payload 주입 가능 → origin 화이트리스트가 필수 방어선
- **signed URL TTL**: 너무 짧으면 이미지 많을 때 만료, 너무 길면 유출 위험 — 5분 + 재발급 흐름 필요

**중립**
- `publicationPackage.js` 의 클립보드 복사 경로는 fallback 으로 유지 (확장 미설치 사용자)
- WorkflowStatus 무변경 — 발행은 도메인 바깥 사용자 행위

**대안 (rejected)**
- **서버 측 headless browser 자동 발행**: 약관 위반 + 봇 탐지 + 2FA + 형사 리스크
- **네이버 공식 OpenAPI**: 신규 발급 중단으로 접근 불가
- **public URL 본문 삽입**: 외부 호스팅 의존 + 네이버가 자기 CDN 복사 안 함 → 이미지 깨짐 위험
- **데스크톱 앱(Electron + BrowserView)**: webview 로 SmartEditor 접근은 가능하나, OS 별
  빌드·서명·자동 업데이트 파이프라인이 추가로 필요해 확장 대비 유지보수 비용이 불균형.
  설치 마찰도 큼. 브라우저 안에서 동작하는 게 SmartEditor 접근에 자연스러움
- **클립보드 복사 유지만**: 이미지 미전달이라는 근본 결함 미해결

**관련 ADR / 컴포넌트**
- `momently_console` `publicationPackage.js`: 현재 클립보드 발행 패키지 — 이 ADR 의 blocks 빌더로 확장
- ADR 003 (ID strategy): signed URL 의 리소스 식별에 사용
- 이미지 storage(MinIO/R2 등): signed URL + CORS 정책 결정 필요 (구현 7c 단계)
