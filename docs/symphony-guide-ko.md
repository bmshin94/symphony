# Symphony 한국어 분석 & 실전 가이드

OpenAI가 공개한 [Symphony](https://github.com/openai/symphony)를 분석하고, 설치·사용법부터
수익화 아이디어와 다른 언어로의 이식 가능성까지 정리한 문서입니다.

- 원본 저장소: <https://github.com/openai/symphony>
- 이 문서가 있는 포크: <https://github.com/bmshin94/symphony>
- 라이선스: Apache License 2.0

---

## 목차

1. [Symphony란 무엇인가](#1-symphony란-무엇인가)
2. [저장소 구조](#2-저장소-구조)
3. [동작 흐름](#3-동작-흐름)
4. [설치 방법](#4-설치-방법)
5. [WORKFLOW.md 작성법](#5-workflowmd-작성법)
6. [실행 방법](#6-실행-방법)
7. [문제 해결](#7-문제-해결)
8. [코드에서 확인한 빈틈](#8-코드에서-확인한-빈틈)
9. [수익화 아이디어](#9-수익화-아이디어)
10. [React / PHP로 만들 수 있을까](#10-react--php로-만들-수-있을까)
11. [참고 링크](#11-참고-링크)

---

## 1. Symphony란 무엇인가

한 줄 요약: **이슈 트래커를 계속 감시하다가, 티켓이 올라오면 자동으로 격리된 작업 폴더를 만들고
코딩 에이전트(Codex)를 띄워서 끝까지 처리하게 만드는 상주 서비스**입니다.

README의 핵심 문장:

> 코딩 에이전트를 **감독**하는 게 아니라, **일**을 관리하게 해준다.

### 기존 방식 vs Symphony

| 기존 방식 | Symphony |
|---|---|
| 사람이 AI 옆에 붙어서 지시 → 확인 → 재지시 | 할 일 목록에 티켓만 등록 |
| 한 번에 한 작업 | 최대 10개 작업 동시 진행 |
| 작업 맥락이 세션마다 휘발 | 이슈별 워크스페이스가 보존됨 |

### 비유

- 주문서 = 이슈 트래커의 티켓
- 요리사 = 코딩 에이전트(Codex)
- **매니저 = Symphony** ← 이 프로그램

매니저가 주문을 확인하고, 요리사에게 전용 주방(격리된 폴더)을 배정하고, 레시피(프롬프트)를 건네고,
끝날 때까지 지켜보다가, 실패하면 다시 시키고, 완료되면 주방을 정리합니다.

### 지원하는 이슈 트래커

Linear, GitHub Issues, Jira Cloud, Asana, GitLab (총 5종 어댑터 내장)

---

## 2. 저장소 구조

```
symphony/
├── SPEC.md          # 2,312줄 명세서 (이 프로젝트의 본체)
├── README.md        # 소개 및 실행법
├── elixir/          # Elixir/OTP 참조 구현체
├── .codex/skills/   # 코딩 에이전트용 작업 매뉴얼 7종
├── .github/         # CI 워크플로우 + 릴리즈 자동화
└── docs/            # 문서 (이 파일 포함)
```

### 2.1 `SPEC.md` — 진짜 주인공

<https://github.com/openai/symphony/blob/main/SPEC.md>

`MUST` / `SHOULD` / `MAY` (RFC 2119) 어투로 작성된 언어 중립 명세서입니다. 총 15개 챕터.

| 섹션 | 내용 |
|---|---|
| §3 | 8개 컴포넌트 구조 (Orchestrator, Workspace Manager, Agent Runner 등) |
| §7 | 상태 머신 (`Unclaimed` → `Claimed` → `Running` → `Released`) |
| §8 | 폴링 / 스케줄링 / 재시도 백오프 |
| §9 | 워크스페이스 격리 및 안전 불변식 |
| §10 | 코딩 에이전트 실행 프로토콜 |
| §11 | 이슈 트래커 어댑터 계약 |
| §15 | 보안 및 운영 안전 |

README가 제시하는 첫 번째 사용법이 인상적입니다:

> **Option 1. 직접 만들어라** — 좋아하는 코딩 에이전트에게 `SPEC.md` URL을 주고
> 원하는 언어로 구현시켜라.

즉 이 저장소는 **완성품이 아니라 설계도**에 가깝습니다.

### 2.2 `elixir/` — 참조 구현체

<https://github.com/openai/symphony/blob/main/elixir/README.md>

주요 모듈:

| 파일 | 역할 |
|---|---|
| `orchestrator.ex` | 폴링 루프, 디스패치, 재시도를 총괄하는 두뇌 |
| `agent_runner.ex` | Codex 프로세스 실행 및 스트리밍 수신 |
| `workspace.ex` + `path_safety.ex` | 이슈별 폴더 격리 및 경로 안전장치 |
| `codex/app_server.ex` | Codex App Server 프로토콜 통신 |
| `linear/`, `github/`, `jira/`, `asana/`, `gitlab/` | 트래커 어댑터 5종 |
| `symphony_elixir_web/` | Phoenix LiveView 대시보드 + JSON API |
| `ssh.ex` | 원격 SSH 워커 지원 |

**왜 Elixir인가?** (FAQ 기준)

- Erlang/BEAM/OTP가 장시간 프로세스 감독(supervision)에 강함
- 실행 중인 서브에이전트를 중단하지 않고 핫 코드 리로딩 가능

### 2.3 `.codex/skills/` — 에이전트용 작업 매뉴얼

<https://github.com/openai/symphony/tree/main/.codex/skills>

| 스킬 | 역할 |
|---|---|
| `commit` | 세션 히스토리를 근거로 깔끔한 커밋 작성 |
| `push` | 브랜치 푸시 + PR 생성/갱신 |
| `pull` | `origin/main`을 머지 방식으로 반영 (rebase 아님) |
| `land` | PR 충돌 해결 → CI 통과 대기 → squash 머지 |
| `linear` | Linear GraphQL 직접 호출 |
| `debug` | 멈춘 run의 로그 추적 |
| `release` | 버전 범프 → 태그 → Burrito 릴리즈 검증 |

> 이 스킬 파일들은 Symphony를 돌리지 않더라도 다른 프로젝트에 그대로 복사해 쓸 수 있습니다.

---

## 3. 동작 흐름

```
트래커에 "Todo" 티켓 생성
        ↓ (기본 30초 주기 폴링)
Orchestrator가 후보 발견 → 클레임(중복 디스패치 방지)
        ↓
<workspace.root>/<이슈ID>/ 폴더 생성
        ↓ (after_create 훅)
git clone + 의존성 설치
        ↓
해당 폴더 안에서 Codex를 app-server 모드로 실행
        ↓
WORKFLOW.md 프롬프트 주입 → 에이전트 작업 시작
        ↓
커밋 → 푸시 → PR 생성 → 티켓을 "Human Review"로 이동
        ↓
사람이 승인하고 "Merging"으로 변경
        ↓
land 스킬이 CI 통과를 확인하고 머지 → "Done"
        ↓
Symphony가 워크스페이스 정리
```

### 워크스페이스 규칙 (SPEC §9)

- 경로: `<workspace.root>/<workspace_key>`
- 이슈별로 완전히 분리된 폴더 사용
- 성공해도 자동 삭제하지 않음 (재사용 목적)
- 티켓이 터미널 상태가 되면 그때 정리

---

## 4. 설치 방법

### 4.1 준비물

| 준비물 | 용도 | 확인 |
|---|---|---|
| Codex CLI | 실제 코딩을 수행하는 에이전트 본체 | `codex --version` |
| git | 워크스페이스에 코드 체크아웃 | `git --version` |
| mise | Elixir/Erlang 버전 관리 | `mise --version` |
| 트래커 토큰 | 할 일 목록 조회 | GitHub 토큰 / Linear 키 등 |

Codex는 로그인까지 완료되어 있어야 합니다.

mise 설치:

```bash
curl https://mise.run | sh
```

### 4.2 방법 A — 미리 빌드된 실행파일 (권장)

[Burrito](https://github.com/burrito-elixir/burrito)로 빌드된 단독 실행파일이
[릴리즈 페이지](https://github.com/openai/symphony/releases)에 올라옵니다.
Erlang과 Elixir가 내장되어 있어 런타임 설치가 필요 없습니다.

지원 타깃: `macos_arm64`, `macos_x86_64`, `linux_arm64`, `linux_x86_64`

```bash
chmod +x ./symphony-v0.0.2-macos_arm64
./symphony-v0.0.2-macos_arm64 --help
```

단, `codex` / `git` / 트래커 자격증명은 여전히 필요합니다.

### 4.3 방법 B — 소스 빌드

```bash
git clone https://github.com/openai/symphony
cd symphony/elixir

mise trust
mise install                  # Erlang 28, Elixir 1.19.5-otp-28

mise exec -- mix setup        # deps.get
mise exec -- mix build        # escript.build → bin/symphony 생성
```

검증:

```bash
mise exec -- make all         # 빌드 + 포맷 검사 + 린트 + 커버리지 + dialyzer
```

---

## 5. WORKFLOW.md 작성법

`WORKFLOW.md`는 **YAML 프론트매터(설정) + 마크다운 본문(에이전트 프롬프트)** 구조입니다.
저장소에 포함된 원본 예시는 Linear 기준입니다:
<https://github.com/openai/symphony/blob/main/elixir/WORKFLOW.md>

### 5.1 GitHub Issues 기준 최소 예시

Linear은 `Rework` / `Human Review` / `Merging` 커스텀 상태를 만들어야 해서,
처음 시작할 때는 GitHub Issues 어댑터가 더 간단합니다.

```bash
export GITHUB_TOKEN=ghp_...
```

```markdown
---
tracker:
  kind: github
  provider:
    repo: "your-org/your-scratch-repo"
    token: $GITHUB_TOKEN
  active_states:
    - open
  terminal_states:
    - closed
polling:
  interval_ms: 30000
workspace:
  root: ~/symphony-workspaces
hooks:
  after_create: |
    git clone --depth 1 https://github.com/your-org/your-scratch-repo .
agent:
  max_concurrent_agents: 1
  max_turns: 10
codex:
  command: codex app-server
---

너는 GitHub 이슈 {{ issue.identifier }} 를 처리하고 있다.

제목: {{ issue.title }}
설명: {{ issue.description }}
URL: {{ issue.url }}

작업 규칙:
1. 사람이 지켜보지 않는 자동 세션이다. 사람에게 후속 작업을 요청하지 말 것.
2. 이 워크스페이스 폴더 안에서만 작업할 것. 다른 경로는 건드리지 말 것.
3. 작업이 끝나면 이슈에 결과를 댓글로 남기고 이슈를 닫을 것.
4. 진짜 막혔을 때(권한/인증 부재)만 중단하고 이유를 이슈 댓글에 남길 것.
```

> 처음에는 반드시 실제 프로젝트가 아닌 **연습용 저장소**로, `max_concurrent_agents: 1`로 시작하세요.

### 5.2 주요 설정값과 기본값

`elixir/lib/symphony_elixir/config/schema.ex` 기준입니다.

| 항목 | 기본값 | 의미 |
|---|---|---|
| `polling.interval_ms` | `30000` (30초) | 트래커 폴링 주기 |
| `workspace.root` | `/tmp/symphony_workspaces` | 워크스페이스 생성 위치 |
| `agent.max_concurrent_agents` | `10` | 동시 실행 에이전트 수 |
| `agent.max_turns` | `20` | 이슈당 최대 연속 턴 수 |
| `agent.max_retry_backoff_ms` | `300000` (5분) | 재시도 백오프 상한 |
| `codex.command` | `codex app-server` | 에이전트 실행 명령 |
| `codex.thread_sandbox` | `workspace-write` | 워크스페이스만 쓰기 허용 |
| `codex.turn_timeout_ms` | `3600000` (1시간) | 턴 스트림 무응답 허용 시간 |
| `codex.stall_timeout_ms` | `300000` (5분) | 멈춤 판정 시간 |
| `hooks.timeout_ms` | `60000` (1분) | 훅 스크립트 제한시간 |
| `server.host` | `127.0.0.1` | 대시보드 바인딩 주소 |

### 5.3 훅 4종 (SPEC §9.4)

```yaml
hooks:
  after_create:  |   # 워크스페이스 최초 생성 시 (git clone 위치)
  before_run:    |   # 매 실행 직전
  after_run:     |   # 매 실행 직후
  before_remove: |   # 워크스페이스 삭제 직전
```

실패 시 동작이 다릅니다:

- `after_create` / `before_run` 실패 → **작업 중단** (fatal)
- `after_run` / `before_remove` 실패 → 로그만 남기고 무시

### 5.4 자주 놓치는 설정

- 네트워크를 쓰는 명령(`npm install` 등)이 있으면 `codex.turn_sandbox_policy`에
  `networkAccess: true`를 넣어야 DNS가 막히지 않습니다.
- 토큰은 절대 리터럴로 쓰지 말고 `$GITHUB_TOKEN` 형태로 참조하세요.
  그래야 Symphony가 에이전트 자식 프로세스 환경변수에서 토큰을 제거해 줍니다.
- 경로 값의 `~`는 홈 디렉터리로 확장됩니다.

---

## 6. 실행 방법

### 6.1 필수 확인 사항 — README에 없는 플래그

`elixir/lib/symphony_elixir/cli.ex`를 보면, 아래 플래그 없이는 **실행 즉시 종료**됩니다.

```
╭───────────────────────────────────────────────────────────────╮
│ This Symphony implementation is a low key engineering preview. │
│ Codex will run without any guardrails.                         │
│ SymphonyElixir is not a supported product and is presented     │
│ as-is.                                                         │
╰───────────────────────────────────────────────────────────────╯
```

의도적으로 길게 만든 확인 플래그입니다:

```
--i-understand-that-this-will-be-running-without-the-usual-guardrails
```

### 6.2 CLI 사용법

```
Usage: symphony [--logs-root <path>] [--port <port>] [path-to-WORKFLOW.md]
```

| 옵션 | 설명 |
|---|---|
| (경로 생략) | 현재 폴더의 `./WORKFLOW.md` 사용 |
| `--logs-root <경로>` | 로그 저장 위치 (기본 `./log`) |
| `--port <번호>` | 웹 대시보드 활성화 (기본은 비활성) |

### 6.3 권장 실행 명령

```bash
./bin/symphony ./WORKFLOW.md \
  --port 4000 \
  --i-understand-that-this-will-be-running-without-the-usual-guardrails
```

### 6.4 관찰 인터페이스

| 경로 | 설명 |
|---|---|
| `/` | Phoenix LiveView 실시간 대시보드 |
| `/api/v1/state` | 전체 런타임 상태 JSON |
| `/api/v1/<issue_identifier>` | 특정 이슈 상태 |
| `POST /api/v1/refresh` | 강제 새로고침 |

로그 확인:

```bash
tail -f ./log/*.log
```

---

## 7. 문제 해결

| 증상 | 원인 및 해결 |
|---|---|
| 빨간 경고 박스 후 종료 | `--i-understand-...` 플래그 누락 |
| `Workflow file not found` | WORKFLOW.md 경로 오류 |
| 부팅 자체가 실패 | YAML 문법 오류 (탭 대신 스페이스 사용) |
| 설정 변경이 반영되지 않음 | 리로드 실패. 마지막 정상 설정으로 계속 동작하며 로그에 에러 기록 |
| 에이전트가 아무 동작도 안 함 | 이슈 상태가 `active_states`에 없음 |
| `npm install` 등 실패 | `networkAccess: true` 미설정 |
| `after_create` 훅에서 clone 실패 | git 인증 문제 (SSH 키 / HTTPS 토큰) |
| Codex 시작 실패 | `codex` 로그인 미완료 |

디버깅 절차는 `.codex/skills/debug/SKILL.md`에 정리되어 있습니다.

### 운영 시 주의사항

1. 실제 프로젝트에 바로 붙이지 말고 연습용 저장소로 먼저 검증할 것
2. `max_concurrent_agents: 10` × `max_turns: 20` = 최대 200턴이 자동 실행될 수 있음 (비용 주의)
3. 토큰은 `$VAR` 참조로만 사용할 것
4. 대시보드는 기본 `127.0.0.1` 바인딩. 인증 기능이 없으므로 외부에 노출하지 말 것
5. 무인 야간 실행 전에 반드시 한 번은 지켜보면서 돌려볼 것

---

## 8. 코드에서 확인한 빈틈

참조 구현체를 직접 확인해 정리한 미구현 영역입니다. 확장 지점이기도 합니다.

| # | 빈틈 | 근거 |
|---|---|---|
| 1 | **인증 없음** | `symphony_elixir_web/router.ex`에 인증 플러그가 전혀 없음. 대시보드/API 무방비 |
| 2 | **영속 저장소 없음** | SPEC §2.1이 "DB 없이 동작"을 명시. 재시작 시 blocked 맵과 스케줄러 상태 소실 |
| 3 | **비용 상한 없음** | `docs/token_accounting.md`는 토큰을 *측정*만 함. 예산 초과 시 정지 기능 없음 |
| 4 | **Codex 전용** | 단, SPEC §10이 프로토콜을 잘 정의해 두어 다른 에이전트로 교체 가능 |
| 5 | **가드레일 없음** | CLI가 확인 플래그를 강제할 정도 |
| 6 | **단일 호스트 중심** | SSH 워커가 있으나 기본적인 수준 |

---

## 9. 수익화 아이디어

### 9.1 라이선스 확인

Apache License 2.0이므로 **상업적 이용이 가능**합니다.

| 허용 | 의무 |
|---|---|
| 유료 판매 | `LICENSE` + `NOTICE` 파일 포함 |
| 클로즈드 소스 파생물 | 수정 사항 명시 |
| 수정 후 자체 제품화 | **OpenAI 상표/브랜드 사용 금지** |
| SaaS 제공 | (특허 조항 포함되어 비교적 안전) |

> "Symphony"라는 이름은 동명의 핀테크 기업이 상표를 보유하고 있으므로 별도 브랜드를 권장합니다.

### 9.2 난이도별 아이디어

**1단계 — 즉시 시작 가능 (비용 거의 없음)**

- 에이전트 오케스트레이션 주제의 한국어 콘텐츠 (현재 자료가 매우 적음)
- 도입 컨설팅. 단 README가 명시하듯 [harness engineering](https://openai.com/index/harness-engineering/)이
  되어 있어야 효과가 나므로, **"harness 정비 컨설팅"** 자체가 더 큰 상품
- `WORKFLOW.md` 템플릿 팩 판매 (프레임워크별/목적별)

**2단계 — 오픈코어 (개발 2~3개월)**

무료 코어 + 유료 애드온으로, 8절의 빈틈을 그대로 상품화합니다.

| 무료 | 유료 |
|---|---|
| 오케스트레이터, 대시보드, 트래커 어댑터 | 인증/권한 관리, DB 영속화, 예산 상한 및 비용 리포트, 감사 로그, **멀티 에이전트 지원** |

멀티 에이전트 지원이 특히 유망합니다. SPEC §10이 에이전트 실행 프로토콜을 명확히
정의해 두었기 때문에 다른 코딩 에이전트 어댑터를 붙일 수 있고,
**벤더 중립 오케스트레이터**는 OpenAI가 직접 만들 유인이 없는 영역입니다.

**3단계 — 버티컬 "작업 공장" (추천)**

플랫폼 대신 반복 작업 하나를 대행하는 서비스:

- 보안 취약점 자동 패치
- 의존성 업그레이드 (React 17→19 등)
- 테스트 커버리지 확보
- 레거시 마이그레이션 (jQuery→React, Python 2→3)
- 문서화

과금은 월 구독보다 **머지된 PR 건당 과금**이 고객 리스크를 낮춥니다.

**4단계 — SaaS 호스팅**

인프라 비용이 크고, OpenAI가 직접 제품화할 가능성이 높으며,
Devin·Cursor 백그라운드 에이전트·GitHub Copilot 등 경쟁이 치열합니다. 개인 개발자에게는 비권장.

### 9.3 핵심 인사이트

오케스트레이터 자체는 해자가 아닙니다. 설계도가 공개되어 있어 누구나 만들 수 있습니다.

SPEC §3.2의 계층 구조를 보면 답이 보입니다:

| 계층 | 제공 주체 | 가치 |
|---|---|---|
| 오케스트레이터 | OpenAI가 무료 공개 | 낮음 |
| **정책 계층 (`WORKFLOW.md`)** | **비어 있음 (각 팀 소유)** | **높음** |
| **검증 / 작업 증명 계층** | **비어 있음** | **높음** |

README의 데모 설명은 에이전트가 "CI 상태, PR 리뷰 피드백, 복잡도 분석, 워크스루 영상"으로
**작업 증명(proof of work)** 을 제공한다고 말합니다.
"AI가 한 일을 어떻게 신뢰할 것인가"는 아직 미해결 문제입니다.

> 결론: **엔진을 팔지 말고, 정책과 신뢰를 팔 것.**

### 9.4 리스크

1. OpenAI의 자체 제품화 가능성 ("engineering preview"는 제품의 전조)
2. 스펙 변경 가능성 (Draft v1)
3. 모델 가격 변동 시 결과 기반 과금의 원가 리스크
4. 경쟁 심화
5. AI 코딩 자동화에 대한 시장의 피로감

---

## 10. React / PHP로 만들 수 있을까

### 10.1 결론

| 언어 | 가능 여부 | 설명 |
|---|---|---|
| React 단독 | 불가 | 브라우저 UI 라이브러리. 장시간 데몬/프로세스 실행 불가 |
| Node.js + TypeScript | **최적** | React와 같은 생태계. 서브프로세스·스트리밍 처리가 매우 자연스러움 |
| PHP | 조건부 가능 | 일반적인 요청/응답 모델로는 불가. CLI 데몬 또는 큐 워커 방식 필요 |

### 10.2 구현에 필요한 능력 (SPEC §10 기준)

1. 장시간 상주 프로세스
2. 자식 프로세스 실행 — `bash -lc <codex.command>`, cwd는 워크스페이스
3. 스트림 처리 — 라인당 최대 10MB 버퍼 권장, stdout(프로토콜)과 stderr(진단) 분리
4. 동시 처리 — 에이전트 수만큼 프로세스와 스트림 리더 유지
5. 다중 타이머 — 폴링, 턴 타임아웃, 스톨 감지, 재시도 백오프
6. HTTP 클라이언트(트래커) 및 선택적 HTTP 서버(대시보드)

### 10.3 React의 올바른 자리

SPEC §3.1의 7번 컴포넌트 `Status Surface`는 **OPTIONAL**이고,
§2.2는 특정 대시보드 구현을 강제하지 않는다고 명시합니다.

```
┌─────────────────┐
│  React 대시보드  │
└────────┬────────┘
         │ GET /api/v1/state  (이미 구현되어 있음)
┌────────▼────────┐
│  오케스트레이터   │
└─────────────────┘
```

기존 Phoenix LiveView 대시보드를 React로 교체하는 것은 완전히 허용됩니다.

### 10.4 Node.js 구현 스케치

```javascript
import { spawn } from "node:child_process";
import readline from "node:readline";

function launchAgent(workspacePath, command) {
  const child = spawn("bash", ["-lc", command], {
    cwd: workspacePath,
    stdio: ["pipe", "pipe", "pipe"],
  });

  // SPEC §10.3: 프로토콜 스트림과 진단 stderr를 분리해서 처리
  const rl = readline.createInterface({ input: child.stdout });
  rl.on("line", (line) => handleUpdate(JSON.parse(line)));
  child.stderr.on("data", (d) => log.debug(d.toString()));

  return {
    send: (m) => child.stdin.write(JSON.stringify(m) + "\n"),
    stop: () => child.kill(),
  };
}
```

추천 스택: Node + TypeScript + Fastify / SQLite 또는 PostgreSQL / React + Vite / SSE

### 10.5 PHP 구현 스케치

php-fpm 요청 모델로는 불가능하며, **CLI 데몬**으로 작성해야 합니다.

```php
$proc = proc_open(
    ['bash', '-lc', 'codex app-server'],
    [0 => ['pipe', 'r'], 1 => ['pipe', 'w'], 2 => ['pipe', 'w']],
    $pipes,
    $workspacePath
);

stream_set_blocking($pipes[1], false);

while (true) {
    $read = [$pipes[1], $pipes[2]];
    $w = $e = null;
    if (stream_select($read, $w, $e, 1)) {
        foreach ($read as $stream) {
            $line = fgets($stream);
            if ($line !== false) {
                handleUpdate(json_decode($line, true));
            }
        }
    }
}
```

**Laravel을 쓰면 8절의 빈틈이 자연스럽게 해소됩니다:**

| 원본의 빈틈 | Laravel 기본 제공 |
|---|---|
| 인증 없음 | Breeze / Sanctum |
| DB 없음 | Eloquent + 마이그레이션 |
| 큐 관리 | Queue + Horizon |
| 스케줄링 | Task Scheduler |

권장 구조:

```
Scheduler (매분) → 트래커 폴링 → Job을 큐에 투입
        ↓
Horizon 워커 N개 (워커 1개 = 이슈 1개 = 에이전트 1마리)
        ↓
React 대시보드 (Inertia.js)
```

워커 프로세스 1개를 에이전트 1마리에 매핑하면 PHP의 동시성 제약을 우회할 수 있습니다.
`memory_limit`를 넉넉히 잡고(10MB 라인 버퍼 대비), `supervisor`로 워커를 관리하세요.

### 10.6 방법별 비교

| | Node + React | Laravel + React | React만 (엔진은 Elixir 유지) |
|---|---|---|---|
| 난이도 | 낮음 | 보통 | 가장 낮음 |
| 예상 기간 | 3~5주 | 4~7주 | 3~5일 |
| 사용 언어 수 | 1개 | 2개 | 2개 |
| 스트리밍 처리 적합도 | 상 | 중 | - |
| DB/인증 기본 제공 | 직접 구현 | 기본 제공 | - |

### 10.7 권장 로드맵

**1주차** — 엔진은 그대로 두고 React 대시보드부터 만들기.
`--port`로 서버를 켜고 `/api/v1/state`를 소비하면 됩니다. 데이터 구조와 상태 흐름을
빠르게 체득할 수 있습니다.

**2~4주차** — MVP 엔진 직접 구현. SPEC 전체를 구현하지 말고 아래로 한정:

- GitHub Issues 어댑터 1종
- 폴링 + 클레임(중복 방지)
- 워크스페이스 생성 + `after_create` 훅
- 에이전트 실행 + 스트림 수신
- 상태 영속화 (SQLite)

SSH 워커, 트래커 5종, 정교한 재시도 정책은 후순위. 이 범위면 1,000~1,500줄 수준입니다.

**5주차 이후** — 9절의 수익화 방향(특히 멀티 에이전트 어댑터)으로 확장.

---

## 11. 참고 링크

### Symphony

- 원본 저장소: <https://github.com/openai/symphony>
- 명세서 `SPEC.md`: <https://github.com/openai/symphony/blob/main/SPEC.md>
- Elixir 구현체 README: <https://github.com/openai/symphony/blob/main/elixir/README.md>
- 워크플로우 예시: <https://github.com/openai/symphony/blob/main/elixir/WORKFLOW.md>
- 에이전트 스킬 모음: <https://github.com/openai/symphony/tree/main/.codex/skills>
- 릴리즈 (사전 빌드 실행파일): <https://github.com/openai/symphony/releases>
- 이 문서가 있는 포크: <https://github.com/bmshin94/symphony>

### 관련 도구 및 문서

- Codex App Server 프로토콜: <https://developers.openai.com/codex/app-server/>
- Harness engineering: <https://openai.com/index/harness-engineering/>
- Burrito (단독 실행파일 빌더): <https://github.com/burrito-elixir/burrito>
- mise (런타임 버전 관리): <https://github.com/jdx/mise>
- 데모 영상: <https://player.vimeo.com/video/1186371009?h=5626e4b899>
