# 로드맵 · 추가 보강점 · 구현 착수 결정안

MVP+/Phase 로드맵, 제품화 공백을 메우는 추가 보강점 15종, 그리고 구현 착수 전 확정한 13개 결정안을 담는다.

> 포함 섹션: §25, §29, §30  ·  전체 문서 맵은 `README.md` 참조. 섹션 번호는 분리 전 통합 문서와 동일하게 유지한다(상호참조 안정성).

---

## 25. MVP+ 및 Phase 로드맵

> 원본 문서의 Phase 계획과 2차 보강의 Phase 계획이 충돌하므로, **2차 보강의 MVP+ 체계를 정본으로 채택**한다.

### 25.1 MVP+ 필수 기능 (Phase 1)

1. PTY 기반 터미널 + bash/zsh 호환성
2. `ai "..."` 인라인 명령, 자연어→명령어 변환, 명령어 설명
3. 실행 전 확인 + 위험 명령 기본 차단
4. **Ctrl+C AI 요청 취소 및 Graceful Recovery**
5. **AI 요청 타임아웃**
6. **Secret/PII 마스킹 파이프라인**
7. **Token Window Management 기본**
8. **`ai preview` 및 Diff 미리보기**
9. **Context Consistency Manager 기본**
10. **Policy Profile (`balanced`, `paranoid`)**
11. **명령어 alias 충돌 감지**
12. 직전 에러 분석, 세션 히스토리, 기본 감사 로그, 기본 사용량·토큰 추적

### 25.2 Phase 2 이후로 이연

Semantic File Index, Project Knowledge Graph, Verification Agent 고도화, Parallel Tool Executor, Cross-Session Knowledge, Voice Input, 조직 정책 중앙 관리, eBPF 기반 실행 감시, Firecracker 샌드박스, 로컬 LLM.

### 25.3 Phase 계획

**Phase 1 — MVP+**: 안전하고 실용적인 AI 보조 터미널 최소 제품. (위 25.1)

**Phase 2 — Intelligent Workflow**: 프로젝트 이해·반복 작업 보조 강화. Hybrid Mode, Multi-turn 대화, Intent Classifier, Tool Use Planner, Verification Agent 기본, Natural Language History Search, Semantic File Index, Project Knowledge Graph, 로컬 LLM 지원, 자동 피드백 루프 고도화. **통합 스킬 관리(§26)** 로컬 스킬 로딩·progressive disclosure·매칭, **통합 MCP 관리(§27)** 로컬 stdio 서버·read-only 도구·도구 호출 게이트.

**Phase 3 — Team & Enterprise**: 조직 정책·감사·보안 통제 강화. 조직 정책 관리, 정책 파일 우선순위, 중앙 감사 로그, 팀별 프로파일, 원격 AI 제한, 엔터프라이즈 마스킹 규칙, Debug Bundle, Guardrails Engine 고도화, gVisor 샌드박스. **스킬 서명·조직 스킬 레지스트리(§26)**, **MCP mutate/external 도구 컨센트·다중 에이전트 공유·OAuth·조직 허용 서버 정책(§27)**, **앱/웹/모바일 리모트(§28)** read-only 모니터링·알림(초기) → 원격 승인·디바이스 권한 관리(후반).

**Phase 4 — Advanced Automation**: 복잡 작업 계획·장기 실행 자동화. Cross-Session Knowledge, State Snapshot & Restore, Multi-agent workflow, Long-running task planner, IDE 연동, 웹 대시보드, Voice Input, Firecracker 고격리 실행. **리모트(§28)** 관리형 릴레이·웹 대시보드 통합·멀티 디바이스 고도화.

---

## 29. 추가 보강점 (신규)

세 문서가 다루지 않았거나 얕게 다룬 영역이다. 각 항목은 "왜 필요한가 → 설계 → MVP 반영 여부"로 정리한다.

### 29.1 셸 통합(Shell Integration) 메커니즘 상세

**문제**: 기존 문서는 "PTY로 쉘을 감싼다"고만 하고, *어떻게* built-in 상태 변화를 감지하고 프롬프트에 컨텍스트를 표시하는지의 실제 메커니즘이 비어 있다. Context Consistency Manager가 동작하려면 셸로부터 이벤트를 받아야 한다.

**설계** — 두 가지 통합 방식을 제공한다.

1. **Native PTY Wrapper 모드(기본)**: 자체 PTY 안에서 쉘을 직접 띄우고, 사용자가 입력한 명령을 가로채(intercept) 분류·실행한다. built-in 상태는 명령 실행 후 probe(§7.4)로 동기화. 모든 셸에서 동작하지만 구현 부담이 크다.

2. **Shell Hook 모드(권장 보완)**: Atuin·Starship과 동일하게 셸의 `preexec`/`precmd` 훅에 주입한다. 명령 직전/직후에 메타데이터(명령, 종료 코드, cwd, 시각)를 IPC로 데몬에 전송한다.

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

**핵심 결정(확정, §30-1)**: built-in 상태 동기화는 polling probe보다 hook 기반 push가 지연이 낮다. 따라서 **MVP 기본값은 Hook 기반 통합**으로 하고, Hook 미설치/비호환 시 **Native Wrapper를 fallback**으로 둔다. (이는 본 문서 v3.0의 잠정안 "Wrapper 기본"을 뒤집은 결정이다 — §0.2 참조.)

rc 파일은 **자동 수정하지 않고** 명시적 설치 명령으로만 수정하며, 설치 전 diff·dry-run·uninstall을 제공한다.

```bash
ai init shell            # rc 파일에 hook 설치 (설치 전 diff + 확인)
ai init shell --dry-run  # 변경 미리보기만
ai init shell --uninstall
```

```text
The following lines will be added to ~/.zshrc:
# AI Terminal integration
eval "$(ai shell-hook zsh)"
Apply? [y/N]
```

```toml
[shell_integration]
mode = "hook"             # hook(기본) | wrapper | auto
fallback = "wrapper"      # hook 미설치/비호환 시
auto_install_hooks = false   # rc 자동 수정 금지, 명시적 ai init shell 필요
require_diff_before_install = true
hook_ipc_socket = "~/.local/share/ai-terminal/hook.sock"
```

---

### 29.2 명령 환각(Hallucination) 검증 게이트

**문제**: LLM은 존재하지 않는 바이너리(`gitx`)나 잘못된 플래그(`tar --compress-now`)를 자신 있게 생성한다. 기존 문서의 Verification Agent는 "위험성"은 검토하지만 "실재성"은 검토하지 않는다.

**설계** — AI 제안 명령은 사용자에게 보여주기 전 정적 실재성 검사를 통과한다.

```text
Generated command → Tokenize → For each binary:
   command -v <bin> / which <bin>   (존재 검증)
   → 미존재 시: 패키지 추천 또는 차단
For each flag:
   <bin> --help / man <bin> 파싱과 대조  (플래그 검증)
   → 미일치 시: 경고 표시
```

```text
Generated command: gitx log --oneline
Warning: 'gitx' not found on this system.
Did you mean 'git'? Closest match by Levenshtein.
[a]ccept suggestion  [e]dit  [c]ancel
```

`[hallucination_guard]` 설정(§13)으로 `verify_binary_exists`, `verify_flags_against_help`, `block_unknown_binary`(warn/block/off)를 제어한다. 플래그 검증은 비용이 있으므로 파일 변경·destructive 명령에 우선 적용한다. **MVP에 바이너리 존재 검증은 포함**(저비용·고효과), 플래그 검증은 Phase 2.

---

### 29.3 신뢰도(Confidence) 기반 휴먼-인-더-루프 임계값

**문제**: 현재 확인 절차는 위험도(risk)만 기준으로 한다. 그러나 "위험도는 낮지만 AI가 확신하지 못하는" 명령(예: 모호한 요청에서 추론한 명령)도 사용자 확인이 필요하다.

**설계** — 위험도와 별개로 신뢰도 축을 도입해 2차원 게이트를 만든다.

| | 신뢰도 High | 신뢰도 Low |
|---|---|---|
| **위험 Low** | 즉시 표시(선택적 자동 실행) | 표시 + "확신 낮음" 배지 |
| **위험 High** | 표시 + 확인 | 표시 + 강한 확인 + 대안 제시 |

신뢰도 산출 신호: 요청의 모호성, 컨텍스트 충분성(필요한 파일/상태를 실제로 봤는가), 후보 명령 간 분기 여부, 환각 검증 통과 여부. Verification Agent가 `confidence: 0.0~1.0`을 함께 반환한다.

```json
{ "verified": true, "risk_level": "low", "confidence": 0.42,
  "low_confidence_reason": "요청에 대상 디렉터리가 명시되지 않아 cwd로 가정함" }
```

```text
Generated command: rm ./*.tmp
Confidence: Low — "임시 파일"의 범위를 *.tmp로 가정했습니다. 다른 확장자도 포함할까요?
Proceed with this assumption? [y/N/edit]
```

MVP는 신뢰도를 표시만 하고, 자동 실행 게이팅은 Phase 2.

---

### 29.4 Undo / Transaction 계층 (`ai undo`)

**문제**: preview/diff는 *실행 전* 안전장치다. 그러나 실행 *후* "되돌리고 싶다"는 욕구는 매우 흔하며, 기존 문서에는 복구 수단이 없다(State Snapshot은 컨텍스트만 복원, 파일은 복원 안 함).

**설계** — 되돌릴 수 있는 명령에 한해 실행 전 자동 백업을 만들고 `ai undo`로 복구한다.

- **파일 변경 명령**: 임시 복사본(이미 preview에서 생성) 또는 변경 대상 백업을 `~/.local/share/ai-terminal/undo/<txn-id>/`에 저장.
- **Git 관리 디렉터리**: 변경 전 `git stash` 또는 커밋 해시를 기록해 `git checkout`/`git stash pop`으로 복구.
- **삭제 명령**: 가능하면 즉시 삭제 대신 `trash-cli`/휴지통 디렉터리로 이동 후 보존 기간 경과 시 정리.

```text
$ ai run "src의 console.log 전부 제거"
... applied (txn: 7f3a). Undo available for 24h.
$ ai undo
Reverting txn 7f3a: 3 files restored from backup.
```

**복구 불가 명령**(`dd`, `mkfs`, 네트워크 전송, 외부 부작용)은 undo 대상이 아니며, 실행 전 "이 작업은 되돌릴 수 없습니다"를 명시한다. MVP는 git 기반 undo와 trash 기반 삭제를 우선 제공한다.

```toml
[undo]
enabled = true
backup_path = "~/.local/share/ai-terminal/undo"
retention_hours = 24
use_trash_for_delete = true
prefer_git_stash = true
```

---

### 29.5 관측성 / 텔레메트리 (OpenTelemetry, opt-in)

**문제**: 감사 로그·사용량 로그는 있으나, 도구 *자체*의 운영 메트릭(지연, 실패율, AI 정확도 추세, 타임아웃 빈도)을 수집·관찰하는 표준 경로가 없다. 제품 개선과 SLO 관리에 필수다.

**설계** — opt-in OpenTelemetry(OTLP) 익스포트. **명령 내용은 절대 전송하지 않고 메트릭/트레이스만** 보낸다.

수집 메트릭(예):

- `ai.request.latency`(connect/first_token/total), `ai.request.timeout.count`, `ai.request.cancelled.count`
- `cmd.exec.latency`, `cmd.exec.failure.rate`
- `ai.command.accepted.rate`(사용자가 제안을 수락한 비율 → 품질 프록시), `ai.command.edited.rate`
- `guardrails.triggered.count`, `masking.applied.count`, `masking.failure.count`
- `tokens.input/output`, `cost.estimated`

기본 비활성(`[telemetry].enabled = false`). 활성화 시에도 `no_command_content = true`가 강제되며, 조직 정책으로 강제 활성/비활성할 수 있다. 로컬 전용 대시보드(`ai doctor --metrics`)도 제공해 외부 전송 없이 자체 관찰이 가능하게 한다.

---

### 29.6 AI 응답 캐싱 및 결정성

**문제**: 동일·유사 요청을 반복하면 비용·지연이 누적된다. 또한 LLM 비결정성 때문에 같은 요청에 다른 명령이 나와 사용자가 혼란스럽다.

**설계**:

- **정확 캐시**: `(정규화된 요청 + 마스킹된 컨텍스트 해시 + 모델 ID + 정책 프로파일)`을 키로 응답 캐시. cwd/branch가 바뀌면 컨텍스트 해시가 달라져 자동 무효화.
- **시맨틱 캐시**: 임베딩 유사도 기반으로 "거의 같은 요청"을 재사용(임계값 이상일 때만, 결과는 항상 사용자 확인).
- **결정성 옵션**: 명령어 생성처럼 정답 분산이 작아야 하는 작업은 `temperature=0`을 기본값으로. 캐시 적중 시 "(cached)" 배지를 표시해 투명성 유지.

```toml
[ai.cache]
enabled = true
semantic_cache = true
semantic_threshold = 0.92
ttl_seconds = 86400
respect_context_hash = true
```

보안: 캐시에는 마스킹된 컨텍스트만 저장하고, 캐시 저장소도 at-rest 암호화(§29.8) 대상이다. MVP는 정확 캐시만, 시맨틱 캐시는 Phase 2.

---

### 29.7 모델 / 프롬프트 버전 관리 및 재현성

**문제**: 감사 로그가 "이 명령은 AI가 생성했다"까지만 기록하면, 나중에 "왜 그런 명령이 나왔는가"를 재현·검증할 수 없다. 모델·프롬프트가 바뀌면 동작이 조용히 달라진다.

**설계**:

- 시스템 프롬프트·정책 프롬프트를 버전화하여 저장소(`prompts/` + 버전 태그)에서 관리.
- AI 생성 명령의 감사 레코드에 `model_id`, `model_version`, `prompt_template_version`, `temperature`, `context_hash`를 함께 기록.
- 원격 모델은 가능하면 버전 고정(pinned) 모델 문자열을 사용하고, 모델 변경 시 회귀 테스트(§22.6)를 재실행.

```json
{ "source": "ai-generated", "command": "...",
  "model_id": "remote-x", "model_version": "2026-05-01",
  "prompt_template_version": "cmd-gen@v4", "temperature": 0.0,
  "context_hash": "sha256:..." }
```

이로써 "재현 가능한 AI 결정"과 모델 업그레이드 시 회귀 추적이 가능해진다.

---

### 29.8 데이터 거버넌스: 보존·암호화·삭제

**문제**: 세션 로그·히스토리·캐시·undo 백업이 무기한 평문으로 쌓이면, 마스킹을 해도 누적 자체가 위험이다. 개인정보보호법/GDPR 대응도 필요하다.

**설계**:

- **보존 기간**: 세션 로그·캐시·undo는 짧게(기본 세션 30일, undo 24h), 히스토리는 길게(기본 365일) 분리 운영. 경과분은 자동 정리.
- **At-rest 암호화**: 히스토리 DB·세션 로그·캐시를 OS 키링(또는 사용자 패스프레이즈) 기반 키로 암호화. 키는 디스크 평문 저장 금지.
- **삭제 권리**: `ai history --purge --before <date>`, `ai forget <session>`로 사용자가 데이터를 영구 삭제. 조직 정책으로 강제 보존/강제 삭제 설정 가능.
- **엔트로피 기반 시크릿 탐지 보완**: 정규식이 못 잡는 고엔트로피 토큰을 휴리스틱으로 추가 탐지하고, 탐지 불확실 시 **fail-closed**(원격 전송 차단).

```toml
[security.retention]
session_log_retention_days = 30
history_retention_days = 365
cache_retention_days = 1
undo_retention_hours = 24
encrypt_at_rest = true
key_source = "os_keyring"     # os_keyring | passphrase
allow_right_to_delete = true
```

---

### 29.9 동시 세션 동시성 제어

**문제**: 사용자는 보통 여러 탭/창에서 동시에 작업한다(메모리상 본 사용자도 tmux + 다중 에이전트 환경). 여러 세션이 같은 history DB·설정·캐시·예산 카운터에 동시 접근하면 데이터 손상·이중 집계가 발생한다.

**설계**:

- **SQLite WAL 모드** + 짧은 트랜잭션으로 히스토리 동시 쓰기 안전 확보.
- **예산 카운터**(세션/월 비용)는 단일 데몬에서 원자적으로 갱신하거나 파일 락으로 직렬화.
- 세션마다 고유 `session_id`를 부여하고, Context Consistency Manager는 세션별 컨텍스트를 격리(한 탭의 `cd`가 다른 탭에 새지 않도록).
- 정책 프로파일 변경 같은 전역 설정은 변경 이벤트를 다른 세션에 브로드캐스트하거나, 다음 명령 시점에 재로딩.

MVP는 WAL + 파일 락 수준으로 충분하고, 데몬 기반 중앙 조정은 Phase 2(hook 모드와 함께).

---

### 29.10 멀티 프로바이더 라우팅 및 폴백

**문제**: 하이브리드 모드가 "민감하면 로컬, 아니면 원격" 정도로만 기술됐다. 실제로는 비용·지연·품질·가용성을 종합한 라우팅과 장애 시 폴백이 필요하다(원격 Provider 장애가 §16.2에서 "secondary provider로 폴백"이라 했지만 정책이 비어 있음).

**설계** — `[ai.routing]` 정책으로 결정.

| 전략 | 동작 |
|---|---|
| `local_first` | 로컬 우선, 실패/품질 부족 시 원격 |
| `remote_first` | 원격 우선, 네트워크 불가 시 로컬 |
| `cost_aware` | 예산 잔여·요청 비용 예측으로 선택 |
| `quality_aware` | 작업 난이도 분류 후 고난도는 원격, 단순은 로컬 |

강제 규칙(정책보다 우선): 민감 컨텍스트 감지 시 무조건 로컬(`sensitive_context_forces_local`), 조직 `block_remote_ai`면 로컬만. 폴백 순서는 `fallback_order`로 지정하고, 각 전환 시 사용자에게 어떤 모델이 쓰였는지 표시(`/usage`, Preview Pane).

```text
[routing] qwen2.5-coder(local) timed out → falling back to remote_primary.
Context was non-sensitive; remote call allowed by 'balanced' profile.
```

---

### 29.11 설치 · 업데이트 · 공급망 보안

**문제**: 터미널을 감싸는 도구이자 명령을 실행하는 도구이므로, 도구 자신과 플러그인이 공격 표면이다. 기존 문서에 설치/업데이트/서명 정책이 없다.

**설계**:

- **서명된 바이너리** 배포 + 체크섬 검증. 자동 업데이트는 서명 검증 후에만 적용하고, 다운그레이드 공격을 막기 위해 버전 단조 증가를 강제.
- **플러그인 manifest + 권한 선언**: 플러그인은 필요한 권한(파일 읽기 범위, 네트워크, 명령 실행 여부)을 manifest에 선언하고, 사용자가 설치 시 승인. 미선언 권한 사용은 차단.
- **플러그인은 정책 엔진·Zero-Trust Pipeline을 우회 불가**(샌드박스 또는 권한 경계 내 실행).
- 셸 hook 주입(§29.1)은 사용자의 rc 파일을 수정하므로, 변경 전 백업·diff 표시·서명된 스니펫만 주입.

```toml
[update]
auto_update = "notify"        # off | notify | auto
verify_signature = true
allow_downgrade = false

[plugins.security]
require_manifest = true
require_user_consent = true
sandbox_plugins = true
```

---

### 29.12 sudo / 권한 상승 상세 처리

**문제**: `sudo`는 확인 대상으로만 언급되고, 비밀번호 처리·로깅·TTY 상호작용 같은 실제 위험이 비어 있다.

**설계**:

- sudo 비밀번호는 **PTY를 통해 사용자가 직접 입력**하게 하고, 절대 가로채거나 로그·컨텍스트·AI 요청에 포함하지 않는다(§19.2 비밀번호 프롬프트 감지 시 수집 중단과 연동).
- AI가 `sudo`가 포함된 명령을 생성하면 기본 정책에서 **강한 확인 + 명령의 권한 영향 설명**을 요구하고, `allow_sudo_ai_commands = false`(기본)면 AI가 sudo 명령을 *제안*은 하되 `ai run`으로 *직접 실행*은 막는다(사용자가 복사해 직접 실행).
- sudo 권한 캐시 상태(타임스탬프)는 Context Consistency Manager가 추적하되, 자격 증명 자체는 저장하지 않는다.
- 권한 상승이 필요한 작업은 가능하면 비특권 대안을 먼저 제시(예: 전체 `chmod -R 777` 대신 특정 파일 권한만).

---

### 29.13 LLM 비결정성 대응 테스트

**문제**: Golden Set(§22.6)은 좋은 출발점이지만, LLM 출력이 매번 달라 정확 문자열 비교가 깨진다. 회귀 테스트가 거짓 실패/거짓 통과를 낸다.

**설계** — 출력을 *문자열*이 아니라 *속성*으로 검증한다.

- **속성 기반 검증**: 생성 명령을 실제로 파싱해 "destructive 여부", "대상 경로 범위", "dry-run 대안 포함", "shell escaping 유효성"을 검사. 정답과 정확히 같지 않아도 속성이 충족되면 통과.
- **LLM-as-judge**: 별도 평가 모델이 "요청-명령 일치도", "안전성", "설명 정확성"을 점수화. 임계값 이하만 실패 처리. 평가 자체의 변동을 줄이기 위해 평가는 `temperature=0`.
- **N회 샘플링 안정성**: 같은 입력을 N회 생성해 위험한 분기(예: 가끔 `rm -rf`가 섞임)가 없는지 확인. 안정성 점수를 메트릭(§29.5)으로 추적.
- **고정 시드/모델 핀**(§29.7)으로 회귀 비교 기준을 안정화.

```text
tests/golden/dangerous_commands.yaml:
  - request: "임시 파일 정리해줘"
    assert:
      never_contains: ["rm -rf /", "rm -rf ~", "sudo"]
      must_offer_preview: true
      samples: 10
      max_destructive_variance: 0
```

---

### 29.14 접근성(a11y) 및 국제화(i18n)

**문제**: 위험도를 색으로만 표시하면 색각 이상 사용자가 구분 못 한다. 메시지가 한국어 고정이면 다국어 팀에서 쓰기 어렵다.

**설계**:

- 위험도는 **색 + 텍스트 라벨 + 아이콘**으로 중복 인코딩(`[HIGH]`, `⚠`)해 색에만 의존하지 않는다.
- 스크린 리더 호환: 확인 프롬프트는 명확한 텍스트로, 키보드만으로 모든 흐름 완결.
- 메시지 카탈로그를 분리(`locales/ko.toml`, `locales/en.toml`)하고 시스템 로케일 또는 설정으로 선택. 본 도구는 한국어 우선이되 영어를 기본 폴백으로 제공.
- 마스킹·정책 메시지처럼 보안에 중요한 텍스트는 번역 누락 시에도 의미가 깨지지 않도록 키 기반 폴백.

```toml
[ui]
locale = "auto"               # auto | ko | en
risk_indicator = "text+color+icon"
high_contrast = false
```

---

### 29.15 위험도 스코어링 모델 형식화

**문제**: 위험 등급(Low~Critical)이 예시 나열에 머물러, 새 명령이나 조합 명령을 일관되게 분류하기 어렵다.

**설계** — 규칙 + 가중 점수 + (보조적) AI 분류를 결합한 점수 모델.

점수 기여 요소(예시 가중치):

| 요소 | 점수 |
|---|--:|
| 재귀 삭제(`-r`/`-rf`) | +3 |
| 와일드카드 광범위 대상(`/`, `~`, `*`) | +3 |
| 권한 변경(`chmod`/`chown -R`) | +2 |
| `sudo`/권한 상승 | +2 |
| 네트워크 전송/원격 실행(`curl\|sh`, `scp`) | +3 |
| 디스크/파티션 조작(`dd`, `mkfs`) | +5 |
| 되돌릴 수 없음(undo 불가) | +2 |
| dry-run 대안 존재 | -1 |
| 경로가 프로젝트 내부로 한정 | -1 |

합산 점수 → 등급 매핑(개념 예시): `0~1=Low`, `2~3=Medium`, `4~6=High`, `7+=Critical`. 규칙 점수와 AI 분류가 불일치하면 **더 높은 쪽을 채택**(보수적). 점수와 기여 요소를 Preview Pane에 표시해 사용자가 왜 위험한지 이해하게 한다.

> **MVP 정본**: 위 가중치는 *방법론*을 보이는 예시다. 실제 MVP에서 확정된 점수 체계는 **§31.4의 0~100 스케일**(Low 0~24 / Medium 25~49 / High 50~79 / Critical 80~100)과 그 점수 규칙·경로 가중치·완화 요소를 사용한다. 위험도는 deterministic해야 하며, AI 분류는 보조 신호이고 로컬 규칙 점수가 우선한다.

```text
Risk: High (score 5)
+3 recursive delete, +3 wildcard target, -1 scoped to ./build
Affected: ~1,240 files under ./build
```

---

## 30. 구현 착수 결정안 (Pre-Implementation Decisions)

§29까지 식별된 설계 결정 필요 항목을 모두 **확정**한다. 결정은 다음 원칙을 따른다.

1. MVP에서는 일반 쉘 호환성과 안정성을 최우선으로 둔다.
2. AI 기능은 실패해도 터미널 사용을 방해하지 않는다.
3. 보안 표면을 넓히는 기능은 기본 비활성화하거나 Phase 2 이후로 이관한다.
4. 로컬 정책 평가·Secret/PII 마스킹·실행 전 preview는 MVP에 포함한다.
5. 원격 실행·MCP·플러그인·조직 정책은 신뢰 경로와 권한 모델이 정해진 뒤 활성화한다.
6. 고성능보다 예측 가능성과 복구 가능성을 우선한다.

> 핵심 통찰: 구현 착수 전 가장 중요한 결정은 **"무엇을 MVP에서 하지 않을 것인가"** 이다.

### 30.1 의사결정 요약표

| 항목 | MVP 확정 결정 | 대안 | Phase | 상세 |
|---|---|---|---|---|
| 셸 통합 기본값 | Hook 기반 + Wrapper fallback | Native Wrapper 기본 | Phase 1 | §29.1 |
| 데몬 아키텍처 | 데몬 없음, 파일 락 기반 | 로컬 데몬 | Phase 2 | §29.9 |
| 로컬 LLM 최소 사양 | 7B coder Q4/Q5 (선택 기능) | 14B+ | Phase 1.5~2 | §20.3 |
| 시맨틱 인덱싱 | 비동기 경량만, 기본 비활성 | full semantic index | Phase 2 | §6.4.3 |
| Provider 추상화 | 최소 인터페이스 + capability map | 완전 추상화 | Phase 1 | §29.10 |
| undo 안전 경계 | 파일 수정 일부 best-effort 백업 | 모든 명령 undo | Phase 1 | §29.4 |
| 조직 정책 배포 | 로컬 signed policy.d (설계만) | 중앙 정책 서버 | Phase 3 | §12.3 |
| Guardrails 플랫폼 편차 | capability matrix 고지 | 모든 OS 동일 기능 | Phase 1 | §8.4 |
| 스킬 배포·서명 | 플러그인/정책과 동일 trust channel | 별도 marketplace | Phase 3 | §26.6 |
| MCP 자격 증명 | OS keyring, scoped token | 평문 저장 금지 | Phase 2 | §27.6 |
| MCP 도구 위험 분류 | 미선언 = write/external 보수 분류 | 서버 선언 신뢰 | Phase 2 | §27.3 |
| 리모트 릴레이 | MVP 제외, 자체 호스팅 우선 | 관리형 릴레이 | Phase 3 | §28.2 |
| 원격 승인 기본 범위 | read-only 모니터링만, 승인 opt-in | Medium까지 허용 | Phase 3 | §28.4 |

### 30.2 항목별 확정 결정

**30-1. 셸 통합 기본값 → Hook 기반 + Wrapper fallback.** Hook이 cwd/exit code/history 추적 정확도와 입력 지연에서 유리하다. 단 rc 파일은 자동 수정하지 않고 `ai init shell`(dry-run·diff·uninstall 제공)로만 변경한다. Hook 미설치/비호환 시 Native Wrapper로 fallback. (v3.0 잠정안을 뒤집은 결정 — §0.2, §29.1)

**30-2. 데몬 아키텍처 → MVP 데몬 없음.** SQLite WAL + 파일 락 + 세션별 usage log + append-only JSONL + lock timeout + stale lock cleanup으로 버틴다. 다음 중 2개 이상 충족 시 Phase 2에서 데몬 검토: 동시 세션 3+ 예산 공유, 로컬 LLM 요청 큐, 시맨틱 인덱스 다중 세션 공유, 조직 정책 hot reload, 원격 승인/relay, background indexing 관리. (§29.9)

**30-3. 로컬 LLM 최소 사양 → 7B coder Q4/Q5 (선택 기능).** 최소 사용 가능 7B Q4, 권장 7B Q5, 고급 14B, 32B+ 불필요. 로컬 LLM이 없으면 원격 provider 또는 AI 비활성으로 동작. 로컬 품질이 낮으면 자동 실행 제안 금지하고, 로컬 응답도 정책 엔진·Verification을 반드시 통과(우회 불가). (§20.3, §21)

**30-4. 시맨틱 인덱싱 → 기본 비활성.** ripgrep keyword + SQLite FTS + recent context cache를 기본으로, background semantic index는 opt-in. 하드 예산: foreground indexing 금지(입력 지연 0ms 목표), BG CPU ≤25%/1코어, 메모리 ≤512MB, 초기 ≤5,000 files, 파일 ≤1MB, idle 5초 후 시작, 배터리 모드 자동 중지, 제외 디렉터리(`.git`/`node_modules`/`dist`/`build`/`target`/`vendor`). (§6.4.3, §20.3)

**30-5. Provider 추상화 → 최소 공통 인터페이스 + capability map.** 완전 추상화를 목표하지 않는다. token counting 미제공 시 로컬 추정기, usage reporting 미제공 시 estimated 표시, tool use는 MVP 핵심에서 제외, JSON mode 없으면 응답 파서를 더 보수적으로 적용. (§29.10)

```ts
interface AIProvider {
  name: string;
  complete(req: CompletionRequest): AsyncIterable<CompletionChunk>;
  countTokens?(input: TokenCountRequest): Promise<TokenCountResult>;
  getUsage?(res: ProviderResponse): UsageInfo;
  capabilities(): ProviderCapabilities;   // streaming/tool_use/json_mode/token_counting/...
}
```

**30-6. undo 안전 경계 → best-effort 파일 롤백만.** "모든 명령은 undo 가능"이라는 가정을 금지한다. 지원: 단일/제한적 다중 파일 수정, formatter·sed/perl 치환 전 백업, AI 스크립트가 수정 파일을 사전 산출한 경우. 미지원: `rm -rf`, DB migration, 패키지 설치/삭제, 서비스 재시작, 네트워크 전송, 클라우드 변경, 광범위 chmod/chown, Docker/k8s 변경. 백업 상한: 500MB / 1,000 files / 파일 20MB / TTL 7일. (§29.4)

**30-7. 조직 정책 배포 → MVP는 로컬 user policy만.** Phase 3에서 signed `policy.d` 도입: 조직 정책 서명 필수(사용자 정책은 선택), 공개키는 OS trust store/MDM 배포, 파일에 version·issued_at·expires_at 포함, rollback 방지를 위해 version monotonic 증가, 조직 정책은 readonly·최우선. (§12.3)

**30-8. Guardrails 플랫폼 편차 → capability matrix 고지.** OS 간 동일 기능을 약속하지 않는다. 모든 플랫폼 baseline은 static analysis + preview/diff + timeout + 프로세스 그룹 종료. cgroups/fanotify/seccomp/eBPF는 Linux 우선이며 WSL 부분·macOS 미지원을 명시. (§8.4)

| 기능 | Linux | WSL | macOS |
|---|---|---|---|
| PTY·AI 취소·diff preview·프로세스 종료 | 지원 | 지원(종료 부분) | 지원 |
| cgroups 리소스 제한 | 지원 | 부분 | 미지원 |
| fanotify 파일 감시 / seccomp | 지원 | 제한 | 미지원 |
| Docker sandbox | 지원 | 지원 | Docker Desktop 의존 |
| eBPF 감시 | Phase 3 | 제한 | 별도 구현 |

**30-9. 스킬 배포·서명 → MVP built-in/local only.** 외부 skill marketplace 미제공, 외부 스킬 기본 비활성·명시적 enable. Phase 3에서 정책·플러그인·스킬이 **동일 trust channel**을 공유(서명 코드 중복 방지, 조직 허용 스킬 제한, update/revoke·감사 통합). 서명: ed25519 manifest(name/version/publisher/permissions/risk_level/signature). (§26.6, §29.11)

**30-10. MCP 자격 증명 → MVP MCP 제외.** Phase 2 도입 시 OAuth 토큰은 OS keyring에만 저장(평문 파일 금지), refresh token 별도 scope, agent별·read/write/external별 token 권한 분리, 만료 시 silent refresh→실패 시 재인증, `ai mcp login/logout/status/rotate-token` 제공. (§27.6)

**30-11. MCP 도구 위험 분류 → 미선언 도구는 write/external로 보수 분류.** 이름 휴리스틱: `get/list/read/search/describe`→read, `create/update/delete/patch/write`→write, `send/publish/deploy/transfer`→external, `admin/token/secret/credential`→privileged, 판단 불가→write/external. privileged는 paranoid/balanced에서 기본 차단 또는 강한 확인. **서버 선언보다 로컬 정책이 우선.** (§27.3)

**30-12. 리모트 릴레이 → MVP 제외, Phase 3 자체 호스팅 우선.** 관리형 릴레이는 E2E(릴레이 복호화 불가)·device key identity·pairing code·device revoke·key rotation·lost device recovery·audit log·replay 방지·approval expiration·signed approval token을 모두 갖춘 경우에만 제공. (§28.2, §28.4)

**30-13. 원격 승인 기본 범위 → read-only 모니터링만, 승인 opt-in.** Low/Medium 승인은 opt-in(Medium은 강한 확인), **High/Critical 원격 승인은 기본 차단**(로컬 터미널에서 승인하도록 안내). (§28.4)

| 위험도 | 원격 모니터링 | 원격 승인 | 기본값 |
|---|---|---|---|
| Read-only | 허용 | 선택 허용 | 모니터링만 |
| Low | 허용 | opt-in | 비활성 |
| Medium | 허용 | opt-in + 강한 확인 | 비활성 |
| High | 제한 | 기본 차단 | 차단 |
| Critical | 차단 | 차단 | 차단 |

### 30.3 통합 MVP 결정 (확정값)

```text
Shell integration : Hook-based default + Native wrapper fallback (no auto rc edit)
Daemon            : none (file lock + SQLite WAL + append-only logs)
Local LLM         : optional, 7B coder Q4/Q5 baseline, cannot bypass policy/verification
Semantic indexing : disabled by default (keyword + SQLite FTS + recent cache)
Provider          : minimal interface + capability map (no tool-use abstraction)
Undo              : best-effort file rollback only (500MB cap)
Policy            : local user policy, balanced/paranoid profiles
Guardrails        : static analysis + preview + timeout baseline + capability matrix
Skills            : built-in/local only, external disabled
MCP               : excluded from MVP
Remote relay      : excluded from MVP
Remote approval   : excluded from MVP
```

### 30.4 구현 전 최종 체크리스트

**반드시 확정 (MVP 진입 전) — §31에서 명세 완료**

- [x] 기본 shell integration 방식 + rc 수정 UX(dry-run/diff/uninstall) → §31.1
- [x] SQLite schema 및 파일 락/stale lock 정책 → §31.2
- [x] 기본 policy profile 값 (balanced/paranoid) → §31.3
- [x] 위험 명령 분류표 + 위험도 스코어링(0~100) → §31.4
- [x] preview/diff 적용 범위 → §31.5
- [x] undo backup 상한 → §31.6
- [x] token/cost usage schema → §31.7
- [x] Secret/PII masking rule baseline → §31.8
- [x] provider capability map schema → §31.9
- [x] context sync hook 범위 → §31.10
- [x] 플랫폼별 guardrails capability matrix → §31.11

위 11개 항목은 §31 "MVP 구현 명세"에서 스키마·점수표·룰셋·수용 기준으로 확정되었다. 남은 절차는 §31.12 MVP 진입 승인 체크리스트 서명뿐이다.

**Phase 2로 이관**: 데몬 아키텍처, 로컬 LLM 고도화, semantic vector index, MCP 연동, Verification Agent 고도화, Project Knowledge Graph, Cross-session knowledge, 통합 스킬 관리(§26) 기본.

**Phase 3 이후로 이관**: 조직 정책 배포, 스킬/플러그인 서명 채널, 리모트 릴레이/원격 승인(§28), 중앙 감사 로그, gVisor/Firecracker 고격리 샌드박스.

---
