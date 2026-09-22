# Coolify CLI 전수조사 & 활용 분석 (한국어 정리본)

> 작성일: 2026-09-22
> 대상 저장소: <https://github.com/bmshin94/coolify-cli>
> 원본(업스트림): <https://github.com/coollabsio/coolify-cli>
> Coolify 본체: <https://github.com/coollabsio/coolify> · <https://coolify.io>
> API 스펙: <https://github.com/coollabsio/coolify/blob/v4.x/openapi.json>

---

## 목차

1. [프로젝트 정체](#1-프로젝트-정체)
2. [폴더 구조 전수조사](#2-폴더-구조-전수조사)
3. [아키텍처와 데이터 흐름](#3-아키텍처와-데이터-흐름)
4. [숨겨진 기능 2개 (v5 메시)](#4-숨겨진-기능-2개-v5-메시)
5. [설치 및 사용법](#5-설치-및-사용법)
6. [플러그인 / 스킬 / MCP 정체 정리](#6-플러그인--스킬--mcp-정체-정리)
7. [API 토큰 정책](#7-api-토큰-정책)
8. [깃허브에서 유명한 이유](#8-깃허브에서-유명한-이유)
9. [로컬 AI 에이전트 구축 활용법](#9-로컬-ai-에이전트-구축-활용법)
10. [React / PHP 로 만들 수 있는가](#10-react--php-로-만들-수-있는가)
11. [수익화 아이디어 9선](#11-수익화-아이디어-9선)
12. [추천 로드맵과 리스크](#12-추천-로드맵과-리스크)

---

## 1. 프로젝트 정체

**한 줄 요약**: Coolify(셀프호스팅 PaaS)를 터미널에서 조종하는 Go 기반 공식 CLI의 포크본.

비유하면 **"내 서버용 Vercel/Heroku를 조종하는 리모컨"**.

| 항목 | 내용 |
|---|---|
| 저장소 | `bmshin94/coolify-cli` (포크) |
| 업스트림 | `coollabsio/coolify-cli` |
| 언어 | Go 1.26 |
| 프레임워크 | Cobra(CLI) + Viper(설정) + xdg(경로) |
| 코드 규모 | Go 코드 **40,689줄**, 파일 **329개** |
| 명령어 수 | **273개** (`llms-full.txt` 기준) |
| 라이선스 | Apache 2.0 |
| 배포 | GoReleaser (Linux/macOS/Windows × amd64/arm64) |
| 테스트 | CI 강제 최소 커버리지 70% |

### Coolify가 뭔지 (배포 방식 비교)

| 방법 | 비유 | 특징 |
|---|---|---|
| Vercel / Heroku | 호텔 | 편하지만 비싸고 커스터마이즈 제한 |
| 생 AWS EC2 | 맨땅에 집짓기 | 저렴하지만 전부 직접 |
| **Coolify** | 내 땅에 조립식 주택 | 내 서버에 설치하면 Vercel 같은 UI가 생김 |

---

## 2. 폴더 구조 전수조사

```
coolify-cli/
├── coolify/main.go          # 진입점 (cmd.Execute() 호출만)
├── cmd/                     # 명령어 레이어 (222개 파일, 30개 폴더)
│   ├── root.go              # 루트 커맨드 + 전역 플래그 + 서브커맨드 등록
│   ├── context/             # 인스턴스(접속 대상) 관리
│   ├── application/         # 앱 관리 (env, storage, tag, task, previews, create)
│   ├── database/            # DB 관리 (backup, env, storage, tag)
│   ├── service/             # 원클릭 서비스 (WordPress 등)
│   ├── server/ project/ deployment/ resources/ destination/
│   ├── github/ gitlab/      # Git 연동
│   ├── s3/ notification/ teams/ settings/ tag/ sharedenv/
│   ├── cloudtoken/ cloudinit/  # Hetzner 등 클라우드 프로비저닝
│   ├── mcp/                 # ★ Coolify 서버 쪽 MCP 기능 on/off
│   ├── init/                # ★ WireGuard 메시 부트스트랩 (미등록)
│   ├── firewall/            # ★ 크로스호스트 방화벽 (미등록)
│   ├── common/              # init/firewall 공용 플래그
│   ├── completion/ config/ update/ version/
├── internal/                # 비즈니스 로직 (107개 파일)
│   ├── api/                 # HTTP 클라이언트 (재시도 3회, 타임아웃 30s)
│   ├── service/             # 서비스 레이어 (도메인 로직)
│   ├── models/              # 응답 구조체
│   ├── output/              # table / json / pretty 포맷터
│   ├── config/              # ~/.config/coolify/config.json
│   ├── cli/ parser/ version/
│   └── ssh/ wireguard/ firewall/ services/   # v5 메시 (8,108줄)
├── test/fixtures/           # 테스트 픽스처
├── scripts/                 # install.sh, install.ps1, e2e-mesh.sh
├── .github/workflows/       # test.yml, release-cli.yml
├── README.md (33KB)
├── ARCHITECTURE.md (18KB)
├── CONTROL_PLANE.md (48KB)  # v5 컨트롤 플레인 설계 스펙
├── CLAUDE.md (44KB)         # AI 에이전트용 지침
└── llms.txt / llms-full.txt (138KB)  # AI가 읽는 명령어 카탈로그
```

### 숫자로 보는 규모

```
40,689줄  ████████████████████  전체 Go 코드
 8,108줄  ████                  그 중 숨겨진 v5 메시 코드 (약 20%)
   273개  명령어
   329개  파일
    70%   CI 강제 최소 테스트 커버리지
```

---

## 3. 아키텍처와 데이터 흐름

### 4단 레이어

```
사용자 입력
   ↓
cmd/                  Cobra: 플래그 파싱, 검증
   ↓
internal/service/     비즈니스 로직
   ↓
internal/api/         HTTP + Bearer 토큰 + 재시도(3회)
   ↓
Coolify 서버          https://<host>/api/v1/...
   ↓
internal/models/      역직렬화
   ↓
internal/output/      table / json / pretty 출력
```

### 카페 비유

| 레이어 | 비유 | 역할 |
|---|---|---|
| `cmd/` | 주문 받는 직원 | "아메리카노 아이스 톨" 주문 접수/검증 |
| `internal/service/` | 바리스타 | 레시피대로 제조 |
| `internal/api/` | 커피머신 | 실제 추출, 고장 시 3번 재시도 |
| Coolify 서버 | 원두창고(본사) | 진짜 데이터 |
| `internal/output/` | 컵에 담기 | 표/JSON 중 원하는 모양으로 |

레이어를 나눠놔서 로직을 바꿔도 명령어 인터페이스는 그대로, HTTP 구현을 바꿔도 로직은 그대로 유지된다.

### 적용된 설계 패턴 (ARCHITECTURE.md 기준)

- 의존성 주입 (Dependency Injection)
- 전략 패턴 (출력 포맷터)
- 옵션 패턴 (API 클라이언트 생성)
- 에러 래핑 (`fmt.Errorf("...: %w", err)`)

### Context = 주소록

`~/.config/coolify/config.json`

```json
{
  "instances": [
    { "name": "prod", "fqdn": "https://coolify.mysite.com", "token": "...", "default": true },
    { "name": "dev",  "fqdn": "https://dev.mysite.com",     "token": "..." }
  ]
}
```

카톡 계정 전환처럼 `coolify context use dev` 로 대상 인스턴스를 바꾼다.

### 중요 규칙: UUID 사용

- 사용자 대면 인자는 **항상 UUID**, 내부 숫자 ID는 쓰지 않는다.
- 예외: **팀(teams) 명령어만 숫자 ID** 사용.
- 모델에서 `ID int` 는 `table:"-"` 태그로 출력에서 숨김.

---

## 4. 숨겨진 기능 2개 (v5 메시)

`cmd/root.go` 에 `AddCommand` 가 없어서 **공개 CLI에는 등록되지 않은** 두 명령어가 존재한다. 테스트는 계속 돌지만 실제 실행은 불가능하다.

### `coolify init` — WireGuard 메시 + Podman 부트스트랩

Coolify API를 **전혀 쓰지 않는 유일한 명령어**. SSH로 원격 호스트에 접속해서 직접 설치한다.

- N대 호스트에 **풀메시 WireGuard 오버레이** 구성
- 호스트마다 관리 IP `/32` (`100.64.0.0/16`, RFC 6598 CGNAT)
- 네임스페이스마다 컨테이너 서브넷 `/24` (`10.210.0.0/16`)
- Podman + `podman.socket` + 브리지 네트워크 생성
- `coolify-mesh-fw.service` 방화벽 스캐폴드 (기본 default-deny)
- v5 에이전트 `coold` + `corrosion` 다운로드/설치

서브커맨드 4개: `plan`(읽기전용) / `bootstrap`(최초 설치) / `extend`(호스트 추가) / `upgrade`(에이전트 버전 갱신)

비유: 서울·부산·제주 컴퓨터를 사설 지하철(WireGuard)로 연결해 "옆방 컴퓨터"처럼 만드는 작업.

### `coolify firewall` — 크로스호스트 허용 규칙 클라이언트

- coold 에이전트의 REST API를 SSH 바운스 방식으로 호출
- `allow` / `revoke` / `list` / `containers`
- 규칙 ID = `sha256(namespace|src|dst|proto|port)[:12]`
- 규칙은 **목적지 호스트**에 설치 (DROP이 발생하는 쪽)

비유: 터널 안에서 누가 누구랑 통신할 수 있는지 통행증 발급. 기본은 전부 차단.

### 알려진 한계

- **크로스호스트 default-deny는 동작함** (wg0 ↔ 브리지 간 FORWARD 통과, 실측 검증됨)
- **같은 호스트 내부(동일 브리지) 격리는 미적용** — Linux + netavark + Ubuntu 24.04 특성상 브리지 L2 트래픽이 iptables FORWARD를 우회. v5는 앱별 podman 네트워크(`--opt isolate=true`)로 해결 예정.

---

## 5. 설치 및 사용법

### 설치 4가지

```bash
# 1) 설치 스크립트 (추천) — Linux/macOS  → /usr/local/bin/coolify
curl -fsSL https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.sh | bash

# 2) Homebrew
brew install coollabsio/coolify-cli/coolify-cli

# 3) Windows PowerShell
irm https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.ps1 | iex
#    관리자 권한 없이: $env:COOLIFY_USER_INSTALL=1; irm ... | iex
#    특정 버전:        $env:COOLIFY_VERSION='v1.0.0'; irm ... | iex

# 4) Go
go install github.com/coollabsio/coolify-cli/coolify@latest
```

### 포크본 직접 빌드

```bash
git clone https://github.com/bmshin94/coolify-cli
cd coolify-cli
go build -o coolify ./coolify     # 빌드
go run ./coolify context list     # 빌드 없이 실행
go test ./internal/... -cover     # 테스트 + 커버리지
golangci-lint run                 # 린트
go fmt ./...                      # 포맷
```

### 초기 설정

```bash
# Coolify 대시보드 → /security/api-tokens 에서 토큰 발급

coolify context set-token cloud <token>                        # 클라우드
coolify context add -d prod https://coolify.mysite.com <token> # 셀프호스팅

coolify context verify     # 연결/인증 확인
coolify context version    # 서버 API 버전
coolify context use dev    # 기본 컨텍스트 전환
```

### 자주 쓰는 패턴

```bash
coolify server list
coolify project list
coolify app list --format json | jq '.[] | .name'

coolify deploy name my-app --force
coolify deploy batch api,worker,frontend --force
coolify deploy cancel <deployment-uuid>

coolify app logs <uuid> --follow --show-timestamps
coolify app env create <uuid> --key API_KEY --value secret
coolify app env sync <uuid> --file .env.production --build-time

coolify database backup list <database-uuid>
coolify service create <type> --project-uuid <uuid> --server-uuid <uuid> --instant-deploy

coolify completion zsh > ~/.zsh/completions/_coolify   # 셸 자동완성
```

### 전역 플래그

| 플래그 | 설명 |
|---|---|
| `--context <name>` | 특정 컨텍스트 사용 |
| `--token <token>` | 설정 파일 토큰 무시하고 직접 지정 |
| `--format table\|json\|pretty` | 출력 형식 |
| `--show-sensitive`, `-s` | 민감값 표시 |
| `--debug` | 디버그 출력 |

---

## 6. 플러그인 / 스킬 / MCP 정체 정리

**결론: 셋 다 아니다. 독립 실행 바이너리(CLI)다.**

| 분류 | 해당 여부 | 설명 |
|---|---|---|
| **CLI 바이너리** | ✅ 정답 | Go로 컴파일된 단일 실행파일 |
| Claude Skill | ❌ | 단, 스킬로 감싸기 매우 쉬움 |
| MCP 서버 | ❌ | CLI 자체는 MCP 서버가 아님 |
| 플러그인 | ❌ | 호스트에 꽂아 쓰는 구조가 아님 |

### MCP와의 관계가 3겹이라 헷갈리는 지점

1. **`coolify mcp enable` 명령어가 있긴 하다** — 하지만 이건 `POST /api/v1/mcp/enable` 을 호출해 **Coolify 서버 쪽 MCP 기능**을 켜는 스위치일 뿐. CLI가 MCP 서버가 되는 게 아니다. (`cmd/mcp/mcp.go` 확인)
2. **`llms.txt` / `llms-full.txt`** — MCP 없이도 AI가 이 CLI를 쓰게 하는 방식. AI가 문서를 읽고 Bash로 `coolify` 명령을 실행한다.
3. **Claude Skill로 감싸기 가능** — `SKILL.md` 작성 후 `llms.txt` 참조시키면 끝.

즉 **AI 친화적으로 설계된 순수 CLI**이며, MCP 래퍼나 Skill 패키징은 별도로 만들어야 한다.

---

## 7. API 토큰 정책

### 일반 명령어 (273개 중 대부분) — 토큰 필수

```
Authorization: Bearer <token>
→ https://<host>/api/v1/applications
```

발급 위치: Coolify 대시보드 → `/security/api-tokens`

| 스코프 | 용도 |
|---|---|
| `read` | 조회만 |
| `write` | 생성/수정/삭제 |
| `write:sensitive` | 이메일 설정 등 민감 작업 (root 팀 필요) |
| `deploy` | 배포 전용 |

### `coolify init` / `coolify firewall` — 토큰 불필요, SSH 키 필요

- Coolify API를 전혀 쓰지 않음
- `--ssh-key` (SSH 개인키 경로) 필수
- firewall은 추가로 coold 베어러 토큰이 필요하지만, 각 호스트가 설치 시 `/etc/coolify/api-token` 에 랜덤 생성하고 CLI가 SSH로 읽어오므로 사용자가 신경 쓸 필요 없음
- `--coold-token` 은 동일 토큰을 쓰는 CI/테스트 환경용 예외 경로

### 토큰 저장과 보안

- `~/.config/coolify/config.json` 에 평문 JSON으로 저장
- `--show-sensitive` 없이는 `********` 로 마스킹
- 관련 보안 패치 이력: `fix(context): prevent list output from exposing API tokens`
- CI 환경에서는 설정 파일 대신 `--token` 플래그 / 환경변수 주입 권장

---

## 8. 깃허브에서 유명한 이유

### Coolify 본체가 스타인 이유

1. **"셀프호스팅 Vercel/Netlify/Heroku"** — 한 문장으로 설명되는 강력한 포지셔닝
2. **탈클라우드 흐름** — Vercel/Heroku 요금 부담 → 내 VPS로 이전
3. **오픈소스 + 무료** (셀프호스팅 무료, 클라우드만 유료)
4. **원클릭 서비스 수백 개** — WordPress, n8n, Supabase, Ghost 등
5. **Docker/Compose 그대로 사용** — 벤더 락인 없음
6. **1인 개발자 성공 서사** (Andras Bacsai) — 커뮤니티 결집
7. **AI 붐** — n8n, Ollama 등 셀프호스팅 수요 폭증

### CLI 저장소 자체의 이유

| 이유 | 설명 |
|---|---|
| 공식 도구 | coollabsio 공식이라 신뢰도 높음 |
| 자동화 수요 | CI/CD 연동에 CLI 필수 |
| AI 에이전트 시대 | `llms.txt` 채택으로 LLM이 즉시 활용 가능 |
| 코드 품질 | ARCHITECTURE.md, 70% 커버리지 강제, GoReleaser — Go CLI 레퍼런스로 인용됨 |
| v5 떡밥 | 48KB 분량 CONTROL_PLANE.md 스펙 공개 |

---

## 9. 로컬 AI 에이전트 구축 활용법

### 아주 큰 도움

**1) `llms.txt` 패턴 차용**

```
llms.txt (5KB)        요약본, 에이전트가 항상 로드
llms-full.txt (138KB) 전체 카탈로그, 필요할 때만 참조
go run ./coolify docs llms   ← 코드에서 자동 생성
```

문서를 코드에서 자동 생성하므로 명령어가 늘어도 AI용 문서가 낡지 않는다. 다른 프로젝트에도 그대로 적용 가능한 핵심 패턴.

**2) 에이전트의 "손발"로 직접 사용**

```
에이전트 → Bash 툴 → coolify app list --format json → 파싱 → 판단 → coolify deploy
```

모든 명령어가 `--format json` 을 지원해 파싱이 쉽다.

**3) `CLAUDE.md` 작성법 참고**

44KB 분량에 아키텍처, 패턴, 금지사항, 체크리스트가 정리돼 있다. "AI에게 코드베이스를 가르치는 법"의 모범 사례.

### 꽤 도움

**4) 에이전트 인프라 = 배포 대상** — Ollama, n8n, Qdrant, LangFuse 모두 Coolify 원클릭 배포 후 `coolify deploy` 로 CI/CD 연결
**5) 코드 구조 벤치마킹** — cmd/service/api 레이어 분리는 에이전트 툴 서버 설계에 그대로 재사용 가능

### 참고용

**6) `internal/ssh/fanout.go`** — 제네릭 병렬 SSH 실행기, 다중 서버 제어에 재활용 가능
**7) `coolify init`** — 에이전트 여러 대를 VPN 메시로 묶는 레퍼런스 구현

### 조합 예시

```
[Claude Code]
   ├─ Skill: "coolify-deploy" (llms.txt 내장)
   ├─ Bash 툴로 coolify 명령 실행
   └─ 결과 JSON 파싱 후 다음 판단
        ↓
   "어제 배포한 거 롤백해줘" → 에이전트가 자동 처리
```

---

## 10. React / PHP 로 만들 수 있는가

### 결론: 그대로 포팅은 비추, 위에 올라타는 제품은 적극 추천

| 언어 | 가능 여부 | 문제점 |
|---|---|---|
| **Go** (현재) | ✅ | 단일 바이너리, 런타임 의존성 0, 5개 플랫폼 크로스컴파일 |
| **React** | ❌ | React는 UI 라이브러리라 CLI를 만들 수 없음 (Ink 쓰면 가능하나 Node 런타임 필요) |
| **Node/TS** | ⚠️ | 가능하지만 `npm i -g` + node_modules 부담 |
| **PHP** | ⚠️ | Symfony Console / Laravel Zero로 가능하나 PHP 런타임 필요 |

```
Go:   coolify (약 12MB 파일 1개) → 바로 실행
Node: coolify.js + node + node_modules(수백 MB)
PHP:  coolify.php + php 8.x + composer
```

CLI는 "다운받아 바로 실행"이 생명이라 Go/Rust가 사실상 표준이다.

### 반대로 이건 확실히 가능하고 오히려 권장

**A. React로 웹 대시보드** — CLI가 호출하는 REST API를 React가 그대로 호출. 포팅이 아니라 "다른 클라이언트"를 만드는 것.

**B. PHP로 백엔드/웹훅** — Coolify 본체가 **Laravel(PHP)** 기반이라 궁합이 좋다.

```php
Http::withToken($token)->get('https://coolify.io/api/v1/applications');
```

**C. React + PHP 풀스택 (수익화의 핵심 기반)**

```
[React 프론트]           배포 대시보드 UI
      ↕
[PHP/Laravel 백엔드]     권한관리, 멀티테넌시, 과금, 스케줄러
      ↕
[Coolify API]            실제 배포
```

**D. Node/TS로 MCP 서버**

```typescript
server.tool("deploy_app", { uuid: z.string() }, async ({ uuid }) => { /* ... */ });
```

### 정리

| 만들 것 | React | PHP | 추천도 |
|---|---|---|---|
| CLI 자체 포팅 | ❌ | ⚠️ | ★ |
| 웹 대시보드 | ✅✅ | ✅ | ★★★★★ |
| 백엔드/웹훅 | ❌ | ✅✅ | ★★★★ |
| MCP 서버 | ⚠️(Node) | ⚠️ | ★★★★ |
| SaaS 서비스 | ✅✅ | ✅✅ | ★★★★★ |

---

## 11. 수익화 아이디어 9선

### TIER 1 — 지금 당장 가능

#### 1. Coolify 통합 관리 SaaS

**문제**: Coolify는 인스턴스 1개 = 대시보드 1개. 고객사 10곳이면 탭 10개.

```
[React 대시보드]   여러 Coolify 인스턴스를 한 화면에
[PHP/Laravel]      토큰 금고, RBAC, 감사로그, 과금
[Coolify API x N]  실제 제어
```

| 항목 | 내용 |
|---|---|
| 스택 | React + Laravel + PostgreSQL |
| 난이도 | ★★★ |
| 기간 | 2~3개월 |
| 수익 모델 | 인스턴스당 $15/월, 팀 플랜 $49/월 |
| 타겟 | 에이전시, MSP, 프리랜서 |
| 예상 | 고객 100곳 × $29 = 월 $2,900 |

핵심 기능: 통합 배포 히스토리 / 크로스 인스턴스 검색 / 다운타임 알림 / 고객사별 읽기전용 뷰

#### 2. Coolify MCP 서버 + Claude Skill 패키지 (최우선 추천)

```
"프로덕션 API 서버 재시작해줘"    → Claude → MCP → Coolify API
"어제 배포 실패한 앱 로그 보여줘"  → 자동 분석 + 수정 제안
```

| 항목 | 내용 |
|---|---|
| 스택 | TypeScript (MCP SDK) |
| 난이도 | ★★ (`llms-full.txt` 덕에 스펙 확보 완료) |
| 기간 | **2~3주** |
| 수익 모델 | 무료 OSS → Pro $9/월 (승인 워크플로, 감사로그) |
| 차별화 | 위험 명령은 사람 승인 필수인 안전장치 |

가장 빠르고, 시장이 비어 있으며, 깃허브 스타가 다른 제품의 유입구가 된다.

#### 3. 한국형 Coolify 매니지드 호스팅

| 플랜 | 가격 | 내용 |
|---|---|---|
| 설치 대행 | 30만원 (1회) | 서버 세팅 + 초기 구성 |
| 베이직 | 월 15만원 | 모니터링 + 백업 + 업데이트 |
| 프로 | 월 50만원 | + 24시간 대응 + 장애복구 |
| 엔터프라이즈 | 월 150만원~ | + 전담 엔지니어 |

차별점: 네이버클라우드/카카오클라우드 지원, 한국어 지원, 카카오톡 알림, 국내 규정 대응
난이도 ★★ (기술보다 영업) / 초기비용 거의 0 / 고객 20곳 × 15만원 = **월 300만원**

### TIER 2 — 중기 (3~6개월)

#### 4. 배포 템플릿 마켓플레이스

```
[React 마켓]   "Next.js + PostgreSQL + Redis 스택" 선택
[PHP 백엔드]   템플릿 → Coolify API 호출 조합
→ 5분 만에 풀스택 환경 완성
```

무료 + 유료 프리미엄($5~29), 제작자 70% / 플랫폼 30%. 난이도 ★★★, 네트워크 효과 기대.

#### 5. AI 배포 어시스턴트 SaaS

```
배포 실패 → 로그 수집 → LLM 분석 → 원인+해결책 → 승인 시 자동 수정
```

로그 이상탐지, 비용 최적화 제안, 스케일링 추천. $29~99/월 + AI 호출량 과금. 난이도 ★★★★, 가장 미래지향적.

#### 6. 교육 콘텐츠

| 상품 | 가격 | 비고 |
|---|---|---|
| 인프런/유데미 강의 | 5~10만원 | "Coolify로 나만의 Vercel 만들기" |
| 전자책 | 3만원 | |
| 기업 워크샵 | 200~500만원/회 | 수익성 최상 |
| 유튜브 | 광고+협찬 | 유입 채널 |

난이도 ★★. 한국어 Coolify 콘텐츠가 거의 없어 선점 가능.

### TIER 3 — 장기 / 고난이도

#### 7. v5 메시 기능 조기 상용화

`coolify init` + `coolify firewall`(8,108줄)을 먼저 상용화. "멀티 클라우드를 하나의 사설 네트워크로" — AWS + Hetzner + 온프레미스를 WireGuard 메시로 통합. 제로트러스트 네트워킹, 노드당 $50/월. 난이도 ★★★★★, 선점 효과와 리스크가 모두 최대.

#### 8. 컴플라이언스 애드온

ISMS-P / 금융권 대응 감사 로그, SOC2 리포트 자동 생성. B2B 단가 최상 (연 2천만원~). 난이도 ★★★★.

#### 9. Coolify 전문 인력 매칭 플랫폼

개발자 매칭 수수료 10~20%.

### 종합 비교

| # | 아이디어 | 난이도 | 기간 | 수익성 | 추천 |
|---|---|---|---|---|---|
| 2 | MCP + Skill | ★★ | 2~3주 | 중 | 최상 |
| 3 | 매니지드 호스팅 | ★★ | 즉시 | 중~대 | 최상 |
| 6 | 교육 콘텐츠 | ★★ | 1~2달 | 중 | 상 |
| 1 | 통합 SaaS | ★★★ | 2~3달 | 대 | 상 |
| 4 | 템플릿 마켓 | ★★★ | 3~4달 | 중~대 | 중 |
| 5 | AI 어시스턴트 | ★★★★ | 4~6달 | 대 | 중 |
| 7 | v5 메시 | ★★★★★ | 6달+ | 최대 | 주의 |
| 8 | 컴플라이언스 | ★★★★ | 6달+ | 최대 | 주의 |

---

## 12. 추천 로드맵과 리스크

### 로드맵

```
[1~2개월] MCP 서버 오픈소스 공개 → 깃허브 스타 확보
              ↓ (신뢰도 + 유입)
[2~4개월] 교육 콘텐츠 + 매니지드 호스팅 영업 → 현금흐름 확보
              ↓ (고객 피드백 수집)
[4~8개월] React + PHP 통합 SaaS 개발 → 본게임
              ↓
[8개월~]  AI 어시스턴트 기능 추가 → 차별화
```

무자본 시작 + 점진적 확장 구조라 리스크가 낮다. React/PHP 역량은 4~8개월 구간에서 본격적으로 쓰인다.

### 체크할 리스크

| 항목 | 내용 |
|---|---|
| 라이선스 | Coolify는 Apache 2.0 → 상업적 이용 가능, 저작권 고지 필요 |
| 상표권 | "Coolify" 를 제품명에 직접 쓰지 말 것. "~ for Coolify" 형태 권장 |
| 업스트림 경쟁 | Coolify 팀이 직접 만들 가능성 → 틈새 + 한국 시장 특화로 방어 |
| v5 변동성 | `coolify init` / `firewall` 은 알파이자 미등록 상태. API 변경 가능성 높음 |

---

## 참고 링크

- 이 저장소: <https://github.com/bmshin94/coolify-cli>
- 업스트림 CLI: <https://github.com/coollabsio/coolify-cli>
- Coolify 본체: <https://github.com/coollabsio/coolify>
- 공식 사이트: <https://coolify.io>
- 문서: <https://coolify.io/docs>
- OpenAPI 스펙: <https://raw.githubusercontent.com/coollabsio/coolify/refs/heads/v4.x/openapi.json>
- 저장소 내 문서: `README.md`, `ARCHITECTURE.md`, `CONTROL_PLANE.md`, `CLAUDE.md`, `llms.txt`, `llms-full.txt`
