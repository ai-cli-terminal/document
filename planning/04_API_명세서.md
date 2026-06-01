# 04. 인터페이스 / 명령 명세서 (CLI · 내부 API)
> **프로젝트명**: AI CLI 통합 리눅스 터미널
> **버전**: v1.0
> **작성일**: 2026-06-01
> **기술 스택**: Rust · ratatui · tokio · portable-pty · SQLite (대안: Go)
---

> 본 프로젝트는 REST 웹 API가 아니라 **CLI 터미널**이다. 따라서 본 문서는 두 가지 인터페이스를 권위 있게 정의한다.
> 1. **`ai` CLI 명령 인터페이스** — 사용자가 직접 호출하는 외부 인터페이스.
> 2. **내부 프로그램 인터페이스** — `AIProvider`, Hook IPC, MCP 도구 호출 게이트, Remote Gateway 채널 등 컴포넌트 간 계약.
>
> 본 문서가 CLI 명령의 **정본(authoritative source)** 이다. 상위 설계는 `→ 01-core-design.md §9 참조`, 정책은 `→ 02-security-policy.md §10·§12 참조`, 서브시스템은 `→ 03-subsystems-skill-mcp-remote.md §26~§28 참조`, 결정안은 `→ 05-roadmap-enhancements-decisions.md §29·§30 참조`.

---

## 1. 공통 규약

### 1.1 명령 기본 구조

```bash
ai [command] [options] [input]
```

- `command` — 서브커맨드(`ask`, `cmd`, `run`, `preview`, …). 생략 시 인라인 모드(`ai "..."`)로 해석.
- `options` — `--preview`, `--dry-run`, `--diff`, `--failed` 등 플래그.
- `input` — 자연어 요청 또는 대상 경로. 따옴표로 감싼다.

UI 모드 (`→ 01-core-design.md §6.1 참조`):

| 모드 | 호출 | 설명 |
|---|---|---|
| Shell Mode | (기본) | 일반 리눅스 터미널 |
| AI Inline Mode | `ai "..."` | 쉘 프롬프트에서 접두어로 AI 호출 |
| AI Dedicated Mode | `ai shell` | 대화형 전용 모드 |
| Hybrid Mode | `ai shell` 내부 | AI 대화 + `!cmd` 즉시 실행 혼합 |

### 1.2 트리거 alias와 충돌 감지

- 기본 트리거 명령은 `ai`이며, 보조 alias로 `aish`(AI shell 진입)를 제공한다.
- 설치(`ai init shell`) 시 사용자 환경에 이미 `ai`라는 이름의 바이너리/alias/함수가 존재하면 **alias 충돌을 감지**하고 경고한다(MVP+ 필수 기능, `→ 05 §25.1-11`).

```text
Warning: an existing command 'ai' is already defined (alias → 'aptitude install').
Choose an alternative trigger or remove the conflict.
  [1] use 'aiterm' as trigger   [2] keep 'ai' and shadow existing   [c] cancel
```

- 충돌 미해소 시 설치를 중단한다(`auto_install_hooks = false` 원칙, `→ 05 §29.1`).

### 1.3 Hybrid Mode 내부 제어 명령

`ai shell` 내부에서만 유효한 제어 명령. 외부 CLI 서브커맨드와 구분된다.

| 입력 | 동작 |
|---|---|
| 일반 자연어 | AI 요청으로 처리 |
| `!command` | 일반 쉘 명령으로 즉시 실행 |
| `/command` | AI shell 내부 제어 명령 |
| `Ctrl+C` | 현재 AI 요청 또는 실행 중 명령 취소 |
| `Ctrl+D` | AI shell 종료 |

내부 제어 명령: `/status`(컨텍스트 표시) · `/context`(전송 범위 표시) · `/shell`(쉘 모드 전환) · `/clear`(대화 초기화) · `/policy`(정책 표시) · `/usage`(토큰·비용) · `/exit`.

### 1.4 종료 코드 규약

CLI는 표준 쉘 호환 종료 코드를 따른다. 사용자가 `!cmd`로 실행한 일반 명령은 그 명령의 종료 코드를 그대로 전파한다.

| 코드 | 의미 |
|---|---|
| `0` | 정상 완료 (명령 생성·실행·설명 성공) |
| `1` | 일반 오류 (요청 실패, 생성 실패) |
| `2` | 사용법 오류 (잘못된 인자/옵션) |
| `4` | 정책 차단 (Policy Engine `block`) |
| `5` | 사용자 취소 (확인 거부, `Ctrl+C` 취소) |
| `6` | 환각 검증 실패 (`block_unknown_binary`, `→ 05 §29.2`) |
| `7` | 마스킹 실패로 원격 호출 차단 (fail-closed, `→ 02 §10.4`) |
| `8` | AI 요청 타임아웃 (`→ 05 §25.1-5`) |
| `124` | Guardrails 실행 중 강제 종료 (`→ 01-core-design.md §8.3`) |
| `>128` | 시그널 종료(`128+signal`), 일반 쉘 호환 명령에서 전파 |

취소된 AI 요청은 세션 로그에 `cancelled` 상태로 기록한다(`→ 01 §6.1`).

```json
{ "type": "ai_request", "status": "cancelled", "reason": "user_interrupt", "timestamp": "2026-06-01T11:20:00+09:00" }
```

### 1.5 AI 응답 포맷 (정본)

모든 명령 생성 응답은 다음 5블록 구조를 따른다(`→ 01 §9.5`).

````text
Goal: 사용자 요청 요약
Command:
```bash
find . -type f -name "*.log" -mtime +7
```
Explanation:
- 현재 디렉터리 아래 .log 파일을 찾습니다.
- 마지막 수정일이 7일보다 오래된 파일만 표시합니다.
Risk: Low — 파일을 삭제하지 않고 조회만 합니다.
Next: 실행 Enter / 수정 e / 취소 Ctrl+C
````

- 삭제·변경 명령은 반드시 **영향 범위**를 설명한다.
- 위험도는 색 + 텍스트 라벨 + 아이콘으로 중복 인코딩(`[HIGH]`, `⚠`)하여 색에만 의존하지 않는다(`→ 05 §29.14`).
- 신뢰도(confidence)가 낮으면 "확신 낮음" 배지를 함께 표시한다(`→ 05 §29.3`).

### 1.6 권한 · 위험도 등급 표

권한 모델 (`→ 01 §17`, `→ 02 §10.1`). 권한은 `ReadOnly < FileWrite < ProcessControl < Network < Privileged < Destructive` 순서로 강해진다.

| 권한 | 설명 | 매핑 위험 등급 |
|---|---|---|
| ReadOnly | 파일/로그 조회 | Low |
| FileWrite | 파일 생성·수정·삭제 | Medium |
| ProcessControl | 프로세스 종료·서비스 재시작 | Medium |
| Network | 외부 통신 | High |
| Privileged | sudo·시스템 설정 변경 | High |
| Destructive | 디스크 포맷·대량 삭제 | Critical |

위험도 0~100 스케일 (`→ 05 §29.15` / `§31.4`):

| 등급 | 점수 범위 | 처리 원칙 |
|---|---|---|
| Low | 0~24 | 자동 표시(선택적 자동 실행 후보) |
| Medium | 25~49 | 확인 + preview |
| High | 50~79 | 강한 확인 + 대안 제시 |
| Critical | 80~100 | 기본 차단 |

위험도는 deterministic 해야 하며 **로컬 규칙 점수가 우선**, AI 분류는 보조 신호다. 규칙 점수와 AI 분류가 불일치하면 **더 높은 쪽을 채택**(보수적). 점수와 기여 요소를 Preview Pane에 표시한다.

```text
Risk: High (score 5)
+3 recursive delete, +3 wildcard target, -1 scoped to ./build
Affected: ~1,240 files under ./build
```

### 1.7 정책 게이트 공통 흐름

모든 명령 실행 및 MCP 도구 호출은 다음 게이트를 통과한다(`→ 02 §10.3 Zero-Trust`, `§12.1 Policy Engine`).

```text
입력 → Context 동기화(§7) → Secret/PII Masking(§10.4)
  → Prompt Injection Scan(§19.5) → Hallucination Guard(§29.2)
  → Risk Classification(§29.15) → Policy Engine + 정책 프로파일 게이트(§12)
  → (mutate/high) Consent UI + Preview Pane
  → 실행(PTY/Sandbox) → Guardrails 동적 감시(§8)
  → Audit Log + Before/After Snapshot(§10.5)
```

정책 프로파일별 게이트 강도:

| 프로파일 | confirm_level | 원격 AI | 컨텍스트 범위 표시 | Phase |
|---|---|---|---|---|
| `paranoid` | `all_ai` (모든 AI 명령 확인) | 차단 | 항상 | MVP |
| `balanced` (기본) | `medium_and_above` | 허용 | 민감 시 | MVP |
| `poweruser` | `high_and_above` | 허용 | 미표시 | Phase 2 |
| `dev` | `medium_and_above` + preview 중심 | 허용 | 민감 시 | Phase 2 |

`→ 02 §12.2`에 전체 프로파일 필드값.

---

## 2. 도메인별 CLI 명령 명세

명령 표 공통 형식: **명령 | 설명 | 권한/위험도 | 정책 처리 | Phase**.

### 2.1 핵심 AI 명령

자연어 → 명령 생성/실행/설명/분석을 담당하는 1차 인터페이스(`→ 01 §9.2`).

| 명령 | 설명 | 권한/위험도 | 정책 처리 | Phase |
|---|---|---|---|---|
| `ai ask "<질문>"` | 자연어 질문 답변(명령 생성 아님) | ReadOnly / Low | 컨텍스트 마스킹만, 실행 없음 | 1 |
| `ai cmd "<요청>"` | 자연어 → 쉘 명령 변환(실행 안 함) | 생성 명령에 따름 | 생성·표시만, 실행 게이트 미적용 | 1 |
| `ai run "<요청>"` | 생성 명령 확인 후 실행 | 생성 명령에 따름 | 정책 게이트 전체 + 실행 전 확인 | 1 |
| `ai run --preview "<요청>"` | 실행 전 preview 강제 | 생성 명령에 따름 | preview/diff 후 확인 강제 | 1 |
| `ai preview "<요청>"` | 변경 결과 diff 미리보기(적용은 별도) | FileWrite / Medium | 임시 복사본/dry-run으로 diff 산출 | 1 (MVP+) |
| `ai explain "<명령\|출력>"` | 명령/출력 의미 설명 | ReadOnly / Low | 실행 없음, 컨텍스트 마스킹 | 1 |
| `ai fix [last-error]` | 직전 에러 분석·수정 제안 | 제안 명령에 따름 | 분석은 read, 수정 실행은 게이트 | 1 |
| `ai summarize <파일\|로그>` | 파일·로그·출력 요약 | ReadOnly / Low | 청크 요약 전 마스킹·인젝션 스캔 | 1 |
| `ai review-command "<명령>"` | 실행 전 위험성 분석(점수 표시) | ReadOnly / Low | 위험도 스코어링만, 실행 없음 | 1 |
| `ai script "<요청>"` | 반복 작업 스크립트 생성 | 생성 스크립트에 따름 | 생성·표시, 실행은 별도 게이트 | 2 |

위험 명령 처리 규칙(`→ 02 §10.2`): `sudo` 포함 / `rm -rf` / 루트·홈 전체 대상 / 대량 삭제 / 재귀 권한 변경 / 디스크 포맷 / 원격 코드 다운로드 실행 / 네트워크 전송 / 시크릿 출력 가능성 → 모두 실행 전 확인 요구.

파일 변경 명령은 dry-run 대안을 우선 제안한다(`→ 01 §9.4`):

| 도구 | Dry-run |
|---|---|
| rsync | `--dry-run` |
| git clean | `-n` |
| terraform | `plan` |
| kubectl | `--dry-run=client` |
| npm | 일부 명령 `--dry-run` |
| rm | 직접 없음 → 대상 목록 미리 출력 |

`sudo` 처리(`→ 05 §29.12`): 기본값 `allow_sudo_ai_commands = false`이면 AI가 sudo 명령을 *제안*은 하되 `ai run`으로 *직접 실행*은 막는다(사용자가 복사해 직접 실행). sudo 비밀번호는 PTY를 통해 직접 입력하며 로그·컨텍스트·AI 요청에 포함하지 않는다.

### 2.2 세션 · 기록

| 명령 | 설명 | 권한/위험도 | 정책 처리 | Phase |
|---|---|---|---|---|
| `ai history [--ai-only] [--failed]` | AI/쉘 기록 조회 | ReadOnly / Low | AI 생성 명령은 별도 표시 | 1 |
| `ai history --purge --before <date>` | 보존 기간 경과분 영구 삭제 | FileWrite / Medium | 삭제 권리, 조직 정책 강제 보존 가능 | 1 |
| `ai recall "<질의>"` | 이전 세션 작업 검색(NL) | ReadOnly / Low | 마스킹된 히스토리만 검색 | 2 |
| `ai forget <session>` | 특정 세션 데이터 영구 삭제 | FileWrite / Medium | 삭제 권리(`→ 05 §29.8`) | 1 |
| `ai undo [last]` | 직전 트랜잭션 복구 | FileWrite / Medium | best-effort 파일 롤백만(`→ 05 §29.4`) | 1 |

`ai undo` 안전 경계(`→ 05 §30-6`): 지원 = 단일/제한적 다중 파일 수정, formatter·sed/perl 치환 전 백업, AI 스크립트가 수정 파일 사전 산출. 미지원 = `rm -rf`, DB migration, 패키지 설치/삭제, 서비스 재시작, 네트워크 전송, 클라우드 변경, 광범위 chmod/chown, Docker/k8s 변경. 백업 상한 500MB / 1,000 files / 파일 20MB / TTL 7일.

```text
$ ai run "src의 console.log 전부 제거"
... applied (txn: 7f3a). Undo available for 7 days.
$ ai undo
Reverting txn 7f3a: 3 files restored from backup.
```

### 2.3 정책 · 진단

| 명령 | 설명 | 권한/위험도 | 정책 처리 | Phase |
|---|---|---|---|---|
| `ai policy show` | 현재 정책 프로파일 조회 | ReadOnly / Low | 조회만 | 1 |
| `ai policy set <profile>` | 프로파일 전환(balanced/paranoid) | ReadOnly / Low | 조직 readonly 정책은 완화 불가 | 1 |
| `ai doctor` | 환경·통합 진단 | ReadOnly / Low | 진단만 | 1 |
| `ai doctor --sandbox` | 샌드박스 가용성 진단 | ReadOnly / Low | 진단만 | 1 |
| `ai doctor --mcp` | MCP 서버 헬스체크 | ReadOnly / Low | 진단만 | 2 |
| `ai doctor --metrics` | 로컬 텔레메트리 대시보드(외부 전송 없음) | ReadOnly / Low | 명령 내용 미전송(`→ 05 §29.5`) | 2 |

`poweruser`·`dev` 프로파일은 Phase 2 확장이며, `ai policy set`로 전환 가능하다(MVP에서는 balanced/paranoid만 유효).

### 2.4 셸 통합

MVP 기본값은 **Hook 기반 통합 + Native Wrapper fallback**이다(`→ 05 §29.1`, `§30-1`). rc 파일은 자동 수정하지 않고 명시적 설치 명령으로만 변경한다.

| 명령 | 설명 | 권한/위험도 | 정책 처리 | Phase |
|---|---|---|---|---|
| `ai init shell` | rc 파일에 hook 설치(설치 전 diff + 확인) | FileWrite / Medium | diff·확인 강제, 서명된 스니펫만 주입 | 1 |
| `ai init shell --dry-run` | 변경 미리보기만(파일 미수정) | ReadOnly / Low | 표시만 | 1 |
| `ai init shell --diff` | rc 변경 diff 표시 | ReadOnly / Low | 표시만 | 1 |
| `ai init shell --uninstall` | hook 제거 | FileWrite / Medium | 변경 전 diff·백업 | 1 |
| `ai shell-hook <bash\|zsh\|fish>` | 쉘별 hook 스니펫 출력(`eval` 대상) | ReadOnly / Low | 출력만, 직접 실행 없음 | 1 |
| `ai-terminal hook preexec --cmd <c> --cwd <d>` | preexec IPC 이벤트 전송(쉘 훅이 호출) | 내부 / Low | 메타데이터만, 마스킹 적용 | 1 |
| `ai-terminal hook precmd --exit <n> --cwd <d>` | precmd IPC 이벤트 전송(쉘 훅이 호출) | 내부 / Low | 메타데이터만, 마스킹 적용 | 1 |

설치 UX 예시:

```text
The following lines will be added to ~/.zshrc:
# AI Terminal integration
eval "$(ai shell-hook zsh)"
Apply? [y/N]
```

설정(`→ 05 §29.1`):

```toml
[shell_integration]
mode = "hook"             # hook(기본) | wrapper | auto
fallback = "wrapper"      # hook 미설치/비호환 시
auto_install_hooks = false   # rc 자동 수정 금지, 명시적 ai init shell 필요
require_diff_before_install = true
hook_ipc_socket = "~/.local/share/ai-terminal/hook.sock"
```

### 2.5 서브시스템 (스킬 · MCP · 리모트)

세 서브시스템은 모두 기존 위험도 분류·정책 엔진·Zero-Trust·감사 체계 안에서 동작한다(`→ 03 §26~§28`).

#### 2.5.1 `ai skill` (Phase 2 / 서명·레지스트리 Phase 3)

| 명령 | 설명 | 권한/위험도 | 정책 처리 | Phase |
|---|---|---|---|---|
| `ai skill list [--enabled] [--source user\|project\|org]` | 스킬 목록 | ReadOnly / Low | 조회만 | 2 |
| `ai skill search "<질의>"` | 스킬 의미 검색 | ReadOnly / Low | 조회만 | 2 |
| `ai skill show <name>` | 스킬 상세(SKILL.md) | ReadOnly / Low | 콘텐츠는 Zero-Trust 데이터 | 2 |
| `ai skill install <git-url\|path>` | 스킬 설치 | FileWrite / Medium | 서명 검증·인젝션 스캔 | 2 / 서명 3 |
| `ai skill update [name]` / `remove <name>` | 갱신/제거 | FileWrite / Medium | trust channel 검증 | 2 |
| `ai skill enable <name>` / `disable <name>` | 활성/비활성 | ReadOnly / Low | org readonly 강제 스킬은 비활성 불가 | 2 |
| `ai skill validate <name>` | 스키마·서명·allowed-tools 검증 | ReadOnly / Low | 검증만 | 2 |
| `ai skill pin <name>` / `unpin <name>` | 매칭 강제 지정/제외 | ReadOnly / Low | 충돌 해소(project>user>org) | 2 |

스킬 콘텐츠는 신뢰할 수 없는 데이터로 취급하며, 스크립트 실행은 일반 명령과 동일하게 정책 게이트를 받는다. `paranoid` 프로파일은 `signed` trust 등급만 허용한다(`→ 03 §26.6`).

#### 2.5.2 `ai mcp` (Phase 2 / mutate·external·OAuth Phase 2~3)

MCP 도구 호출은 "AI가 취하는 액션"이며, 일반 쉘 명령과 동일하게 게이팅된다(`→ 03 §27.1`).

| 명령 | 설명 | 권한/위험도 | 정책 처리 | Phase |
|---|---|---|---|---|
| `ai mcp list` / `status` | 서버 목록·상태 | ReadOnly / Low | 조회만 | 2 |
| `ai mcp add <name> --transport stdio --command "..."` | 서버 등록 | FileWrite / Medium | sandbox stdio, allowlist 적용 | 2 |
| `ai mcp remove <name>` / `enable <name>` / `disable <name>` | 서버 관리 | FileWrite / Medium | 조직 정책 차단 가능 | 2 |
| `ai mcp tools [server]` | 집계 도구 목록 + 위험도 | ReadOnly / Low | 위험도 표시 | 2 |
| `ai mcp logs <name>` | 호출 이력(인자·결과 요약) | ReadOnly / Low | 마스킹 적용, 자격증명 제외 | 2 |
| `ai mcp consent <name> --grant-read` | 사전 동의 범위 설정 | ReadOnly / Low | read 도구 자동 허용 범위 | 2 |
| `ai mcp login` / `logout` / `status` / `rotate-token` | OAuth/토큰 수명주기 | Network / High | OS keyring 저장, 평문 금지 | 2~3 |

MCP 도구 위험 분류(`→ 03 §27.3`, 미선언 도구 분류 `→ 05 §30-11`):

| 분류 | 예 | 기본 처리 |
|---|---|---|
| read-only resource | 파일/이슈 조회, 검색 | Low — 자동 허용(paranoid 제외) |
| local mutate | 로컬 파일 쓰기 | Medium — 확인 + preview |
| external write | 원격 이슈 생성, 메시지 전송 | High — 강한 확인 |
| irreversible / side-effect | 결제·배포·메일·삭제 | High/Critical — 강한 확인 + 감사, 원격 승인 금지 |

이름 휴리스틱(미선언 시): `get/list/read/search/describe`→read, `create/update/delete/patch/write`→write, `send/publish/deploy/transfer`→external, `admin/token/secret/credential`→privileged, 판단 불가→write/external. **서버 선언보다 로컬 정책이 우선.**

#### 2.5.3 `ai remote` (Phase 3 — 모니터링 초기 / 승인 후반, 릴레이 Phase 4)

> **본 기능은 시스템에서 가장 보안 민감하다.** 기본값은 보수적이고 활성화는 명시적 옵트인이다(`→ 03 §28.1`).

| 명령 | 설명 | 권한/위험도 | 정책 처리 | Phase |
|---|---|---|---|---|
| `ai remote enable` / `disable` / `status` | 리모트 활성/비활성/상태 | ProcessControl / Medium | 기본 비활성, opt-in | 3 |
| `ai remote pair` | 새 디바이스 페어링(QR/코드) | Network / High | 디바이스별 키 발급, mTLS | 3 |
| `ai remote devices` | 등록 디바이스·권한 목록 | ReadOnly / Low | 조회만 | 3 |
| `ai remote revoke <device-id>` | 디바이스 즉시 폐기 | ProcessControl / Medium | 키 회전 트리거 | 3 |
| `ai remote grant <device-id> --permission approve` | 디바이스 권한 부여 | Privileged / High | 기본 read_only, 승인 권한 opt-in | 3 |

원격 승인 기본 범위(`→ 03 §28.4`, `→ 05 §30-13`): Read-only 모니터링만 기본 허용. Low/Medium 승인 opt-in(Medium은 강한 확인). **High/Critical 원격 승인은 기본 차단**(로컬 터미널에서 승인). 외부 부작용 MCP 도구는 원격 승인 대상에서 기본 제외.

---

## 3. 주요 명령 요청/응답 예시 (터미널 출력)

### 3.1 `ai run` — 자연어 → 명령 생성 후 확인 실행

요청:

```bash
$ ai run "node_modules 삭제"
```

응답(터미널 출력):

```text
Goal: 프로젝트의 node_modules 의존성 디렉터리 삭제
Command:
  rm -rf ./node_modules
Explanation:
- ./node_modules 디렉터리와 그 하위 전체를 재귀 삭제합니다.
- 경로가 현재 프로젝트 내부로 한정되어 있습니다.
Risk: Medium (score 5) — [⚠ MEDIUM]
  +3 recursive delete, +3 wildcard target, -1 scoped to ./node_modules
Affected: ~28,400 files under ./node_modules
Reason: Deletes a large dependency directory, but path is scoped.
Execute? [y/N]
```

거부 시 종료 코드 `5`(사용자 취소).

### 3.2 `ai preview` — 파일 변경 diff 미리보기

요청:

```bash
$ ai preview "config.txt에서 foo를 bar로 치환"
```

응답:

```text
Goal: config.txt 내 'foo'를 'bar'로 치환
Proposed command: sed -i 's/foo/bar/g' config.txt
Preview diff:
--- config.txt
+++ config.txt
-foo_enabled=true
+bar_enabled=true
Risk level: Medium [⚠ MEDIUM]   Apply changes? [y/N]
```

적용 시 변경 전 백업을 `~/.local/share/ai-terminal/undo/<txn-id>/`에 저장하고 `ai undo`로 복구 가능(`→ 05 §29.4`).

### 3.3 `ai fix` — 직전 에러 분석

요청:

```bash
$ ai fix last-error
```

응답:

```text
Goal: 직전 명령 'npm run build' 실패 원인 분석
Diagnosis:
- exit code 1, stderr: "Cannot find module 'typescript'".
- devDependencies에 typescript가 설치되지 않았습니다.
Suggested command:
  npm install -D typescript
Explanation:
- TypeScript 컴파일러를 개발 의존성으로 설치합니다.
Risk: Low [LOW] — 네트워크 통한 패키지 설치(되돌리기 가능).
Next: 실행 Enter / 수정 e / 취소 Ctrl+C
```

`ai fix`는 분석(read)은 자동 수행하되, 제안 명령의 실행은 정책 게이트를 거친다.

---

## 4. 내부 프로그램 인터페이스

CLI 외부 인터페이스 아래에서 컴포넌트 간 계약을 정의한다. CLI 명령은 이 내부 인터페이스를 호출해 동작한다.

### 4.1 AIProvider 인터페이스 (`→ 05 §30-5`)

최소 공통 인터페이스 + capability map. 완전 추상화를 목표하지 않는다. token counting 미제공 시 로컬 추정기, usage reporting 미제공 시 `estimated` 표시, tool use는 MVP 핵심에서 제외, JSON mode 없으면 응답 파서를 더 보수적으로 적용.

```ts
interface AIProvider {
  name: string;
  complete(req: CompletionRequest): AsyncIterable<CompletionChunk>;
  countTokens?(input: TokenCountRequest): Promise<TokenCountResult>;
  getUsage?(res: ProviderResponse): UsageInfo;
  capabilities(): ProviderCapabilities;   // streaming/tool_use/json_mode/token_counting/...
}
```

멀티 프로바이더 라우팅(`→ 05 §29.10`) — `[ai.routing]` 정책으로 결정:

| 전략 | 동작 |
|---|---|
| `local_first` | 로컬 우선, 실패/품질 부족 시 원격 |
| `remote_first` | 원격 우선, 네트워크 불가 시 로컬 |
| `cost_aware` | 예산 잔여·요청 비용 예측으로 선택 |
| `quality_aware` | 고난도는 원격, 단순은 로컬 |

강제 규칙(정책보다 우선): 민감 컨텍스트 감지 시 무조건 로컬(`sensitive_context_forces_local`), 조직 `block_remote_ai`면 로컬만. 폴백 전환 시 사용된 모델을 표시한다.

```text
[routing] qwen2.5-coder(local) timed out → falling back to remote_primary.
Context was non-sensitive; remote call allowed by 'balanced' profile.
```

재현성을 위해 AI 생성 명령의 감사 레코드에 모델·프롬프트 메타를 기록한다(`→ 05 §29.7`):

```json
{ "source": "ai-generated", "command": "...",
  "model_id": "remote-x", "model_version": "2026-05-01",
  "prompt_template_version": "cmd-gen@v4", "temperature": 0.0,
  "context_hash": "sha256:..." }
```

### 4.2 Hook IPC 메시지 (`→ 05 §29.1` / `§31.10`)

쉘의 `preexec`/`precmd` 훅이 명령 직전/직후에 메타데이터를 IPC 소켓(`hook_ipc_socket`)으로 데몬/프로세스에 전송한다. 이벤트는 Context Consistency Manager(`→ 01 §7`)의 상태 동기화 입력이 된다.

쉘별 훅 설치(`ai shell-hook`이 출력):

```bash
# bash: bash-preexec.sh 사용
preexec() { ai-terminal hook preexec --cmd "$1" --cwd "$PWD"; }
precmd()  { ai-terminal hook precmd  --exit $? --cwd "$PWD"; }

# zsh
autoload -Uz add-zsh-hook
add-zsh-hook preexec _ai_preexec
add-zsh-hook precmd  _ai_precmd

# fish
function _ai_preexec --on-event fish_preexec; ai-terminal hook preexec --cmd "$argv"; end
```

IPC 메시지 타입(4종):

| 이벤트 | 시점 | 페이로드 핵심 필드 |
|---|---|---|
| `startup` | 셸 세션 시작 | `session_id`, `shell`, `os`, `cwd`, `hostname`, `user` |
| `preexec` | 명령 실행 직전 | `session_id`, `cmd`(마스킹), `cwd`, `ts` |
| `precmd` | 명령 실행 직후 | `session_id`, `exit_code`, `cwd`, `ts`, `duration_ms` |
| `prompt` | 프롬프트 렌더 직전 | `session_id`, `cwd`, `git_branch`, `context_type` |

`cmd` 등 페이로드는 IPC 전송 전 Secret/PII 마스킹을 거친다. 비밀번호 입력 프롬프트 감지 시 컨텍스트 수집을 중단한다(`→ 01 §19.2`).

```json
{ "event": "precmd", "session_id": "s-9f2a", "exit_code": 1,
  "cwd": "/home/user/app", "duration_ms": 842, "ts": "2026-06-01T11:20:00+09:00" }
```

built-in 명령(`cd export unset alias source set …`)은 외부 프로세스 실행만으로 상태 변화를 알 수 없으므로, 훅 수신 후 Shell State Probe(`→ 01 §7.4`: `pwd / env / alias / set / git rev-parse …`)로 동기화한다. 컨텍스트 불일치 감지 시:

```text
Context mismatch detected.
AI context cwd:   /home/user/project
Actual shell cwd: /home/user/project/backend
Refresh AI context before generating command? [Y/n]
```

세션별 `session_id`로 컨텍스트를 격리해 한 탭의 `cd`가 다른 탭에 새지 않도록 한다(`→ 05 §29.9`).

### 4.3 MCP 도구 호출 게이트 흐름 (`→ 03 §27.4`)

AI(또는 Tool Use Planner)가 MCP 도구를 호출하려 할 때 거치는 내부 계약. 핵심 전제: **MCP 도구 호출 = AI가 취하는 액션** → 일반 쉘 명령과 동일하게 게이팅.

```text
AI requests tool call
   → MCP Manager: resolve server/tool, validate args schema
   → Hallucination guard: 존재하는 tool·필수 인자 검증 (§29.2 준용)
   → Risk classification (§27.3)
   → Policy Engine + 정책 프로파일 게이트 (§12)
   → (mutate/external) Consent UI + Preview Pane (인자·예상 영향 표시)
   → Execute via MCP server (timeout 적용)
   → Result → Zero-Trust 처리(인젝션 스캔·마스킹) → Audit log + before/after snapshot
```

- 읽기 전용 리소스 fetch는 `paranoid`를 제외하면 자동 허용하되, 결과를 컨텍스트에 넣기 전 마스킹·인젝션 스캔을 적용한다.
- `allowed_tools` allowlist가 있으면 그 외 도구는 노출하지 않는다.
- 자격 증명은 `credential_ref`(keyring 참조)로만 보관하고 감사 로그·디버그 번들에서 제외한다.

서버 설정 계약 예시(`→ 03 §27.5`):

```toml
[[mcp.servers]]
name = "github"
transport = "stdio"
command = "github-mcp-server"
enabled = true
allowed_tools = ["list_issues", "get_pull_request", "search_code"]   # allowlist
blocked_tools = ["delete_repo"]
risk_overrides = { create_issue = "medium", merge_pull_request = "high" }
credential_ref = "keyring:github-mcp"            # 평문 토큰 저장 금지
```

### 4.4 Remote Gateway 채널 (`→ 03 §28.2`)

Terminal Daemon(host)과 Client(desktop/web/mobile) 간 안전 채널 프로토콜.

```text
Terminal Daemon (host)  ↔  Channel  ↔  Client (desktop/web/mobile)
```

- **연결 방식**: `direct`(LAN, Tailscale/WireGuard, SSH 터널) 또는 `relay`(인터넷 경유). 어느 경우든 **종단 간(E2E) 암호화**로 릴레이는 평문을 보지 못한다.
- 데몬은 리모트를 사실상 요구한다(데몬 아키텍처 결정 `→ 05 §30-2`).

채널 메시지 종류:

| 방향 | 메시지 | 설명 | 권한 |
|---|---|---|---|
| Daemon → Client | `session.mirror` | 출력 스트리밍(read), 마스킹 적용 | read_only |
| Daemon → Client | `approval.request` | AI/위험 명령 확인 푸시 | approve |
| Daemon → Client | `notification` | 확인 대기·완료·guardrail·예산 초과 | read_only |
| Daemon → Client | `task.status` | 장기 작업 상태·로그 tail·종료 코드 | read_only |
| Client → Daemon | `approval.decision` | 승인/거부/편집(서명된 토큰) | approve |
| Client → Daemon | `input.inject` | 입력 주입 | full |

보안 매트릭스(`→ 03 §28.4`):

| 영역 | 정책 |
|---|---|
| 페어링 | 명시적 디바이스 페어링(QR/일회용 코드), 디바이스별 키 발급 |
| 인증 | mTLS 또는 디바이스 키 + 단기 토큰. 모바일 승인 시 생체인증 게이트 옵션 |
| 암호화 | E2E. 릴레이/중계자는 내용 복호화 불가 |
| 권한 분리 | 디바이스별 `read_only` / `approve` / `full` (기본 `read_only`) |
| 출력 마스킹 | 원격 스트리밍 출력에도 Secret/PII 마스킹 적용 |
| 위험 게이트 | High 명령 원격 승인은 옵션이나 **기본 차단**(`allow_remote_approval_for_high_risk=false`, §30-13), **Critical·외부 부작용 MCP 도구는 원격 승인 기본 금지** |
| 감사 | 모든 원격 액션을 디바이스 ID와 함께 감사 로그 기록 |
| 폐기 | 세션 타임아웃, 디바이스 즉시 revoke, 키 유출 시 회전 |

채널 설정(`→ 03 §28.5`):

```toml
[remote]
enabled = false
mode = "direct"                  # direct | relay
require_device_pairing = true
require_biometric_for_approval = true
default_device_permission = "read_only"   # read_only | approve | full
allow_remote_approval_for_high_risk = false
allow_remote_approval_for_critical = false
allow_remote_approval_for_external_mcp_tools = false
stream_output_masking = true
session_timeout_minutes = 30
```

---

## 5. 상호 참조

- 명령 위험도·실행 전 확인 정책 → `02-security-policy.md §10` 참조
- 정책 엔진·프로파일 전체 필드 → `02-security-policy.md §12` / `§31.3` 참조
- 화면 흐름(시퀀스 다이어그램) → `05_화면_흐름_시퀀스_다이어그램.md` 참조
- 셸 통합·undo·라우팅 결정안 → `05-roadmap-enhancements-decisions.md §29·§30` 참조
- 스킬/MCP/리모트 서브시스템 상세 → `03-subsystems-skill-mcp-remote.md §26~§28` 참조
