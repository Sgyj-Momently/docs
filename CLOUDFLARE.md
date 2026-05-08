# Momently와 Cloudflare Tunnel (Zero Trust)

스택 게이트(nginx)·백엔드는 **`macmini-shared`** Docker 네트워크 안에 두고, Mac mini에서 돌아가는 `cloudflared`(예: `mac-infra` Compose의 `tunnel` 프로필)가 **업스트림**으로 접속합니다.

## 프리픽스·호스트네임 권장

- **별도 호스트네임 한 개**(예: `momently.example.com`). 경로 접두어만 쓰는 방식(`/momently/...`)은 SPA·`/api`/말투 API 프록시를 한 nginx에 두는 현재 설계에는 맞지 않음.
- 콘솔·워크플로 API·말투 API는 브라우저 기준으로 **동일 출처**가 되게 nginx(`momently-gateway`) 뒤에 두었으므로, Cloudflare에서는 **단일 Public Hostname**으로 충분.

## 「Published application routes」만 설정하면 외부에서 접근되나요?

Tunnel로 노출 시 대시보드에서 보는 이름이 **`Published application routes`**(또는 Public Hostnames)이라면, 거기서 하는 일은 다음과 같다.

| 구분 | 설명 |
|------|------|
| 할 일 | `https://momently.example.com` → **Public Hostname**에서 Service type 예: HTTP, URL `http://momently-gateway:80` 처럼 **터널 업스트림**을 게이트로 지정한다. Docker DNS 이름 `momently-gateway`로 접근한다. |

다음이 같이 만족돼야 브라우저에서 접속된다.

1. 해당 라우트에 맞춰 Tunnel용 **DNS**(또는 CNAME)가 터널로 연결되어 있음.  
2. `cloudflared` 컨테이너가 **실행 중**이고 **`macmini-shared`에 참여**해 `momently-gateway`에 도달 가능함.  
3. 필요 시 해당 호스트네임에 **[Access 정책](https://developers.cloudflare.com/cloudflare-one/policies/access/)**(로그인·이메일 제한 등)을 덮어씀. 정책 없이 라우트만 있으면 **공개**(Zero Trust 브라우저 정책에 따라 자동 허용)에 가까움.

오케스트레이터 CORS는 `deploy/.env`의 `MOMENTLY_PUBLIC_ORIGIN`을 게이트에 쓰이는 **`https://` 공개 URL**과 동일하게 맞춘다.

## 제가 대신 Cloudflare에 “설정”해 줄 수 있나요?

**아니요.** Zero Trust·DNS·터널은 귀하의 Cloudflare 계정과 토큰으로만 바꿀 수 있고, 여기서는 계정에 로그인하거나 API를 대리 호출할 수 없습니다. 대신 아래 **값만 복사**해서 대시보드에 넣으면 됩니다.

## 대시보드에서 할 일 (Public Hostname / Published application routes)

UI 문구는 Cloudflare 쪽 업데이트로 조금씩 달라질 수 있습니다. 흐름만 맞추면 됩니다.

### 사전 조건

- `mac-infra`에서 `cloudflared`가 **`macmini-shared` 네트워크**에 붙어 있고, 터널 토큰으로 **Healthy** 상태다.
- `deploy/docker-compose.yml`로 Momently 스택이 같은 **`macmini-shared`**에서 떠 있고, `momently-gateway` 컨테이너가 실행 중이다.

### 추가할 Public Hostname (예시)

| 항목 | 넣을 값 |
|------|---------|
| **Public hostname** | `momently` + 본인 존 (예: `momently.example.com`) |
| **Service type** | `HTTP` (또는 동일 의미의 “HTTP” 옵션) |
| **URL (Origin / Service URL)** | `http://momently-gateway:80` |

- **HTTPS는 Cloudflare 쪽(엣지)에서 종료**되고, 터널 → Docker 구간은 위처럼 **평문 HTTP**로 두는 구성이 일반적이다.
- **경로(Path)** 는 **비운다(빈 칸)**. UI 안내 그대로 «Match all paths» 는 **패턴을 넣지 않는 것**이다.
- **`^/blog`처럼 블로그 전용 정규식을 넣으면 안 된다.** 그렇게 설정하면 `https://momently.tpsg.co.kr/` 나 `/api/...` 요청은 이 라우트에 **전혀 매칭되지 않아** 404·다른 라우트 폴백처럼 보인다. Momently는 `/`(SPA), `/api/…`, `/api/v1/voice-profiles/…` 를 같은 호스트에서 쓴다.

### 메뉴 찾기 (대략적 경로)

1. [Cloudflare 대시보드](https://one.dash.cloudflare.com/) → **Zero Trust**
2. **Networks** → **Tunnels** (또는 **Connectors** → **Cloudflare Tunnel**)
3. Mac mini에서 쓰는 **터널 이름** 클릭
4. **Public Hostname** 또는 **Published application routes** → **Add** / **Add a public hostname**

저장 후 DNS가 터널에 연결되면, 브라우저에서 `https://momently.example.com` 으로 접속 가능해야 한다.

### Momently 쪽 환경과 맞추기

`deploy/.env` (또는 compose 환경)에서 공개 URL과 동일하게:

- `MOMENTLY_PUBLIC_ORIGIN=https://momently.example.com`
- `ALLOWED_ORIGINS=https://momently.example.com`
- `MOMENTLY_CONSOLE_USERNAME=...`
- `MOMENTLY_CONSOLE_PASSWORD=...`
- `MOMENTLY_JWT_SECRET=...` (최소 32바이트 이상, `base64:` 접두어 사용 가능)
- `POSTGRES_PASSWORD=...` (작업 기록과 워크플로 상태 저장용 Postgres 비밀번호)
- `SPRING_PROFILES_ACTIVE=docker,postgres,local-photo-info`

(로컬만 쓸 때는 `http://127.0.0.1:18580` 유지.)

### `momently-gateway` 이름으로 안 붙을 때

같은 Docker 네트워크에서도 프로젝트별 **컨테이너 이름**이 DNS로 잡히는 경우가 있다. 그때는 호스트에서 아래로 실제 이름을 확인한 뒤, Public Hostname의 URL을 그 호스트:80으로 바꾼다.

```bash
docker network inspect macmini-shared --format '{{range .Containers}}{{.Name}} {{end}}'
```

`momently` 관련 게이트 컨테이너 이름이 보이면 `http://<그이름>:80` 으로 시도.

### 스크린샷처럼 설정했는데 안 될 때 체크리스트

1. **Path 필드**  
   - `^/blog` 삭제 → **완전히 비우기** 후 저장.  
   - 루트 `/` 접속이 이 라우트를 타야 한다.

2. **Service URL**  
   - 가능하면 `http://momently-gateway:80` 처럼 **`http://` 포함**. (UI에 스킴만 선택돼 있으면 생략 가능한 경우도 있음.)

3. **`cloudflared` 가 `momently-gateway` 를 DNS로 볼 수 있는지**  
   - 둘 다 **`macmini-shared`** 에 붙어 있는지 확인.  
   - `docker compose ps` 로 터널 컨테이너·Momently 스택이 같은 머신에서 실행 중인지 확인.

4. **앱 env** (`deploy/.env`)  
   - `MOMENTLY_PUBLIC_ORIGIN=https://momently.tpsg.co.kr`  
   - `ALLOWED_ORIGINS=https://momently.tpsg.co.kr`  
   재시작 후 CORS 문제를 줄인다.

## 긴 요청(말투 학습·Ollama)과 504

- **게이트 nginx**의 `/api/v1/voice-profiles` 구간은 Ollama 심층 분석까지 한 번에 걸릴 수 있어, `proxy_read_timeout`을 **600초**까지 두었다. 설정 변경 후에는 `momently-gateway` 이미지를 **`docker compose ... up -d --build momently-gateway`** 로 다시 빌드해야 반영된다.
- **Cloudflare 프록시**를 쓰면(오렌지 구름 등) 엣지·터널 경로에 **약 100초 제한**이 있어, 그보다 오래 걸리는 학습은 여전히 504가 날 수 있다. 그때는 `VOICE_DEEP_ANALYSIS=0`으로 빠른 분석만 쓰거나, 로컬 `http://127.0.0.1:18580`으로 확인한다.

## 대용량 업로드와 413

- 게이트 nginx는 `client_max_body_size 150m`으로 맞춰져 있고, Spring multipart 기본값은 `max-request-size: 140MB`다.
- 콘솔은 `GET /api/v1/uploads/config`로 서버 업로드 제한을 읽어 파일 개수·파일당 용량·전체 용량을 사전 검증한다.
- Cloudflare 앞단에서 413이 나면 Cloudflare 플랜/정책의 업로드 제한에 걸린 것이다. 이 경우 `deploy/.env`의 업로드 제한을 더 낮추거나, 큰 동영상은 로컬 게이트(`http://127.0.0.1:18580`)에서 먼저 검증한다.

## 참고

- 로컬 점검: `deploy/docker-compose.yml`에서 게이트는 `127.0.0.1:18580:80`(기본)으로 바인드.
- Prometheus는 외부에 열리지 않고 `mac-mini-shared`에서 `momently-orchestrator:18080/actuator/prometheus`만 스크랩하는 구성이다.
