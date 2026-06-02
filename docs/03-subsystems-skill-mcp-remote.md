# 서브시스템 — 스킬 · MCP · 리모트

AI 능력·액션을 확장하는 세 서브시스템을 정의한다: 통합 스킬 관리, 통합 MCP 관리, 앱/웹/모바일 리모트. 모두 기존 위험도 분류·정책 엔진·Zero-Trust 파이프라인·감사 체계 안에서 동작한다.

> 포함 섹션: §26, §27, §28  ·  전체 문서 맵은 `README.md` 참조. 섹션 번호는 분리 전 통합 문서와 동일하게 유지한다(상호참조 안정성).

---

## 26. 통합 스킬 관리 (Unified Skill Management)

### 26.1 개요와 위치

**스킬(Skill)** 은 AI가 특정 작업을 더 정확히 수행하도록 하는 **재사용 가능한 지침·자원 번들**이다. 한 스킬은 `SKILL.md`(설명 + 절차 + 제약)와 선택적 스크립트·템플릿·참조 파일로 구성된다.

스킬과 플러그인(§14)의 구분이 중요하다.

| 구분 | 스킬 | 플러그인 |
|---|---|---|
| 레벨 | AI 지침/지식 레벨 | 코드/실행 레벨 |
| 형태 | `SKILL.md` + 자원 | 컴파일/스크립트 코드 + 인터페이스 |
| 작동 | AI 컨텍스트에 주입되어 *생성 품질*을 높임 | 훅으로 *실행 파이프라인*을 확장 |
| 예 | "Spring Boot 프로젝트 구조 컨벤션", "Docker Scout 취약점 최소화 절차" | "git-helper", "log-analyzer" 출력 렌더러 |

스킬 매니저는 AI Service Layer에 위치하며 Context Retriever(§6.4.3)·Token Window Manager(§6.4.1)와 직접 연동한다.

### 26.2 디렉터리 구조와 우선순위

```text
~/.config/ai-terminal/skills/            # 사용자 스킬
  docker-helper/
    SKILL.md          # frontmatter: name, description, version, allowed-tools, signature
    scripts/          # 선택: 스킬이 제공하는 보조 스크립트
    references/       # 선택: 길어서 평소엔 로드하지 않는 참조 문서
./.ai-terminal/skills/                   # 프로젝트 스킬 (저장소에 커밋)
/etc/ai-terminal/skills/                 # 조직 스킬 (readonly 가능)
```

해석 우선순위: **project > user > org** (단, 조직이 `readonly`로 강제한 스킬은 사용자가 비활성화할 수 없음 — §12.3과 동일 원칙).

`SKILL.md` frontmatter 예시:

```yaml
---
name: docker-scout-hardening
description: Dockerfile 보안 패치로 Docker Scout 취약점을 최소화하는 절차. "Dockerfile 보안", "취약점 줄이기", "Scout 점수" 요청 시 사용.
version: 1.2.0
allowed-tools: ["read_file", "run:docker scout *"]   # 이 스킬이 호출 가능한 도구/명령 범위
trust: signed                                        # signed | local | unverified
---
```

### 26.3 Progressive Disclosure (점진적 공개)

전체 스킬 본문을 항상 컨텍스트에 넣으면 토큰이 폭증한다. 따라서 3단계로 공개한다.

```text
[Level 0] 항상: 모든 스킬의 name + description (인덱스, 수십 토큰)
   |  사용자 요청과 의미 매칭
[Level 1] 매칭 시: 해당 SKILL.md 본문 로드
   |  본문이 references/ 를 가리키면
[Level 2] 필요 시: 참조 파일을 추가로 로드 (토큰 예산 내)
```

이는 §6.4.1 Token Window Management의 우선순위 배분과 동일한 예산 안에서 이뤄진다. 동시에 활성화되는 스킬 수는 `max_active_skills`로 제한한다.

### 26.4 매칭과 충돌 해소

- **매칭**: 요청 ↔ 스킬 `description`을 의미 매칭(§6.4.3 Context Retriever 재사용: 키워드 + 임베딩). 매칭 점수와 선택된 스킬을 Preview Pane(§6.1)에 표시해 투명성을 유지한다.
- **충돌**: 같은 도메인에 여러 스킬이 매칭되면 우선순위(project > user > org)와 매칭 점수로 정렬하고, 상위 N개만 활성화한다. 사용자는 `ai skill pin/unpin`으로 강제 지정/제외할 수 있다.

### 26.5 스킬 명령

```bash
ai skill list [--enabled] [--source user|project|org]
ai skill search "도커 취약점"
ai skill show docker-scout-hardening
ai skill install <git-url|path>
ai skill update [name] / ai skill remove <name>
ai skill enable <name> / ai skill disable <name>
ai skill validate <name>          # SKILL.md 스키마·서명·allowed-tools 검증
ai skill pin <name> / ai skill unpin <name>
```

### 26.6 보안 (필수)

스킬은 외부에서 받은 *지침과 실행 코드*를 모두 담을 수 있으므로 공격 표면이다.

- **콘텐츠는 Zero-Trust 데이터**: `SKILL.md`·참조 파일은 신뢰할 수 없는 데이터로 취급하고 프롬프트 인젝션 스캔(§19.5)을 통과시킨다. 시스템 정책·사용자 명령이 스킬 콘텐츠보다 항상 우선한다.
- **allowed-tools 경계**: 스킬은 frontmatter에 선언한 도구/명령 범위만 호출할 수 있다. 미선언 호출은 차단된다.
- **스킬 스크립트 = 일반 명령**: `scripts/` 실행은 Policy Engine(§12)·위험도 분류(§10.1)·실행 전 확인·감사 로그(§10.5)를 그대로 받는다. 자동 실행하지 않는다(원칙 19).
- **서명·신뢰 등급**: 원격/조직 스킬은 서명 검증(§29.11)을 요구할 수 있다. `trust` 등급(signed/local/unverified)을 표시하고, `paranoid` 프로파일은 `signed`만 허용한다.
- **마스킹**: 스킬이 참조하는 파일에 Secret/PII가 있으면 AI 주입 전 마스킹(§10.4).

### 26.7 설정

```toml
[skills]
enabled = true
skill_paths = ["./.ai-terminal/skills", "~/.config/ai-terminal/skills", "/etc/ai-terminal/skills"]
progressive_disclosure = true
auto_match = true
max_active_skills = 3
require_signature_for_remote = true
allowed_trust_in_paranoid = ["signed"]
scan_for_prompt_injection = true
```

### 26.8 단계 반영

로컬 스킬 로딩 + progressive disclosure + 매칭은 **Phase 2(Intelligent Workflow)**. 서명·조직 스킬 레지스트리·중앙 배포는 **Phase 3**. MVP+에서는 스킬 없이도 동작하되, 스킬 디렉터리 스캔/인덱스 골격만 둔다.

---

## 27. 통합 MCP 관리 (Unified MCP Management)

### 27.1 개요

**MCP(Model Context Protocol)** 서버는 AI에게 도구(tools)·리소스(resources)·프롬프트(prompts)를 제공한다. 통합 MCP 관리자는 여러 MCP 서버를 **중앙에서 등록·연결·집계·라우팅**하고, 여러 AI 에이전트(예: Claude Code, Codex CLI, Gemini CLI, Cursor)가 **하나의 MCP 설정을 공유**하도록 한다.

핵심 전제: **MCP 도구 호출은 "AI가 취하는 액션"이다.** 따라서 파일 변경·외부 부작용을 일으키는 도구 호출은 일반 쉘 명령과 동일하게 위험도 분류·실행 전 확인·감사 로그를 받는다(원칙 19).

### 27.2 책임

- **Server Registry**: 서버 등록·디스커버리. transport는 `stdio`·`SSE`·`HTTP(streamable)` 지원.
- **Connection Manager**: 연결 수명주기, 재연결(backoff), 헬스체크(`ai doctor --mcp` 연동, §23.1).
- **Capability Aggregation**: 여러 서버의 tools/resources/prompts를 통합하고 **네임스페이스 충돌**을 `server.tool` 형태로 해소.
- **Auth & Credential**: OAuth 플로우·API 토큰 관리. 자격 증명은 키링 암호화 저장(§29.8), 로그·컨텍스트에 평문 노출 금지(§10.4).
- **Tool Invocation Gating**: 도구 호출을 정책·위험도·컨센트로 게이팅(아래 27.4).
- **Tool Routing**: Agent의 Tool Use Planner(§6.4.2)가 필요로 하는 도구를 적절한 서버로 라우팅.
- **Sandboxing**: 로컬 `stdio` MCP 서버 프로세스를 격리 실행(§6.3 원칙 준용).

### 27.3 MCP 도구 위험 분류

| 분류 | 예 | 기본 처리 |
|---|---|---|
| read-only resource | 파일/이슈 조회, 검색 | Low — 자동 허용 |
| local mutate | 로컬 파일 쓰기, 로컬 상태 변경 | Medium — 확인 + preview |
| external write | 원격 이슈 생성, 메시지 전송 | High — 강한 확인 |
| irreversible / side-effect | 결제, 배포, 메일 발송, 데이터 삭제 | High/Critical — 강한 확인 + 감사, 원격 승인 금지(§28) |

서버별 `risk_overrides`로 개별 도구 위험도를 조정할 수 있다.

### 27.4 도구 호출 게이트 흐름 (핵심)

```text
AI requests tool call
   → MCP Manager: resolve server/tool, validate args schema
   → Hallucination guard: 존재하는 tool·필수 인자 검증 (§29.2 준용)
   → Risk classification (27.3)
   → Policy Engine + 정책 프로파일 게이트 (§12)
   → (mutate/external) Consent UI + Preview Pane (인자·예상 영향 표시)
   → Execute via MCP server (timeout 적용, §16.2)
   → Result → Zero-Trust 처리(아래 27.6) → Audit log + before/after snapshot (§10.5)
```

읽기 전용 리소스 fetch는 `paranoid`를 제외하면 자동 허용하되, 그 결과를 AI 컨텍스트에 넣기 전 마스킹·인젝션 스캔을 적용한다.

### 27.5 명령과 설정

```bash
ai mcp list / status
ai mcp add <name> --transport stdio --command "..." 
ai mcp remove <name> / ai mcp enable <name> / ai mcp disable <name>
ai mcp tools [server]          # 집계된 도구 목록과 위험도
ai mcp logs <name>             # 호출 이력(인자·결과 요약, 마스킹 적용)
ai mcp consent <name> --grant-read    # 사전 동의 범위 설정
```

```toml
[mcp]
enabled = true
config_path = "~/.config/ai-terminal/mcp.json"   # 표준 MCP 설정 포맷 공유
auto_connect = false
require_consent_for_mutate_tools = true
require_consent_for_external_tools = true
sandbox_stdio_servers = true
share_with_external_agents = true                # Claude Code/Codex 등과 동일 설정 공유
scan_tool_results_for_injection = true

[[mcp.servers]]
name = "github"
transport = "stdio"
command = "github-mcp-server"
enabled = true
allowed_tools = ["list_issues", "get_pull_request", "search_code"]   # allowlist
blocked_tools = ["delete_repo"]
risk_overrides = { create_issue = "medium", merge_pull_request = "high" }
credential_ref = "keyring:github-mcp"            # 평문 토큰 저장 금지

[[mcp.servers]]
name = "filesystem-readonly"
transport = "stdio"
command = "fs-mcp --readonly --root ./"
allowed_tools = ["read_file", "list_dir"]
```

### 27.6 보안

- **도구 결과도 Zero-Trust**: 외부 MCP 도구의 응답에 "이전 지시를 무시하라" 류 인젝션이 섞일 수 있다(§19.5). 결과는 데이터로만 취급하고 인젝션 스캔 후 컨텍스트에 넣는다.
- **allowlist 우선**: `allowed_tools`가 있으면 그 외 도구는 노출하지 않는다. 조직 정책으로 서버·도구를 전면 차단할 수 있다(§12.3 `block_patterns` 확장).
- **자격 증명 격리**: OAuth/토큰은 키링 참조(`credential_ref`)로만 보관, 감사 로그·디버그 번들(§23.2)에서 제외.
- **외부 부작용 도구의 원격 승인 금지**: 결제·배포·삭제 도구는 모바일 원격 승인 대상에서 기본 제외(§28.5).

### 27.7 단계 반영

로컬 `stdio` 서버 + read-only 도구 + 기본 게이트는 **Phase 2**. mutate/external 도구 컨센트·다중 에이전트 공유·OAuth는 **Phase 2~3**. 조직 차원의 허용 서버 정책·중앙 자격 증명 관리는 **Phase 3**.

---

## 28. 앱/웹/모바일 리모트 기능 (Remote Control)

### 28.1 개요와 유스케이스

실행 중인 터미널 세션을 **데스크톱·웹·모바일 앱에서 원격 모니터링·제어**한다. 대표 유스케이스:

- 장기 작업(빌드·배포·테스트) 진행 상황 모니터링
- 외출 중 AI 확인 프롬프트를 모바일로 받아 **승인/거부/편집**(human-in-the-loop를 모바일로 이동)
- 확인 대기·작업 완료·guardrail 발동·예산 초과 **푸시 알림**
- 여러 디바이스에서 세션 목록 조회·attach

> **이 기능은 본 시스템에서 가장 보안 민감하다.** 원격에서 명령 실행을 승인할 수 있기 때문이다. 따라서 기본값은 보수적이고, 활성화는 명시적 옵트인이다.

### 28.2 아키텍처

§5의 Remote Access Gateway를 통해 동작한다.

```text
Terminal Daemon (host)  ↔  Channel  ↔  Client (desktop/web/mobile)
```

- **연결 방식**: `direct`(LAN, Tailscale/WireGuard, SSH 터널) 또는 `relay`(인터넷 경유). 어느 경우든 **종단 간(E2E) 암호화**로 릴레이는 평문을 보지 못한다.
- 데몬은 §30(구현 착수 결정안)의 데몬 아키텍처 결정(30-2)과 직접 연결된다 — 리모트는 데몬을 사실상 요구한다.

### 28.3 기능

- **Session mirror**: 출력 스트리밍(read). 입력 주입(write)은 별도 권한.
- **Remote approval**: AI 확인 프롬프트(§6.1)와 위험 명령 확인(§10.2)을 클라이언트로 푸시 → 승인/거부/편집. 결정은 데몬으로 안전 채널을 통해 회신.
- **Notifications**: 확인 대기, 작업 완료, guardrail 발동(§8), 예산 초과(§13 `[ai.usage]`).
- **Task monitoring**: 장기 작업의 상태·로그 tail·종료 코드.
- **Multi-device**: 세션 목록, 디바이스별 권한·세션 만료.

### 28.4 보안 (필수)

| 영역 | 정책 |
|---|---|
| 페어링 | 명시적 디바이스 페어링(QR/일회용 코드), 디바이스별 키 발급 |
| 인증 | mTLS 또는 디바이스 키 + 단기 토큰. 모바일 승인 시 생체인증 게이트 옵션 |
| 암호화 | E2E. 릴레이/중계자는 내용 복호화 불가 |
| 권한 분리 | 디바이스별 `read_only` / `approve` / `full` 권한 (기본 `read_only`) |
| 출력 마스킹 | 원격 스트리밍 출력에도 Secret/PII 마스킹 적용(§10.4) — 화면에 토큰 노출 방지 |
| 위험 게이트 | 기본 read-only 모니터링, **Low/Medium만 opt-in 승인**(§30-13). **High·Critical 원격 승인은 기본 차단**(`allow_remote_approval_for_high_risk=false`, 로컬 터미널에서 승인), **외부 부작용 MCP 도구(§27.3)는 원격 승인 기본 금지** |
| 감사 | 모든 원격 액션을 디바이스 ID와 함께 감사 로그(§10.5)에 기록 |
| 폐기 | 세션 타임아웃, 디바이스 즉시 revoke, rc/키 유출 시 회전 |

### 28.5 명령과 설정

```bash
ai remote enable / disable / status
ai remote pair                 # 새 디바이스 페어링 (QR/코드 표시)
ai remote devices              # 등록 디바이스·권한 목록
ai remote revoke <device-id>
ai remote grant <device-id> --permission approve
```

```toml
[remote]
enabled = false
mode = "direct"                  # direct | relay
relay_url = ""                   # relay 모드일 때만
require_device_pairing = true
require_biometric_for_approval = true
default_device_permission = "read_only"   # read_only | approve | full
allow_remote_approval_for_high_risk = false
allow_remote_approval_for_critical = false
allow_remote_approval_for_external_mcp_tools = false
stream_output_masking = true
session_timeout_minutes = 30

[remote.notifications]
on_confirmation_required = true
on_task_complete = true
on_guardrail_triggered = true
on_budget_exceeded = true
```

### 28.6 단계 반영

원격 **모니터링(read-only) + 알림**은 **Phase 3** 초기. 원격 **승인(approve)** 과 입력 주입(full), 멀티 디바이스 권한 관리는 **Phase 3 후반**. 관리형 릴레이·웹 대시보드 통합은 **Phase 4**(§25.3의 IDE 연동·웹 대시보드와 함께).

---
