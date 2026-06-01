# MVP 구현 명세 (Implementation Spec)

§30.4 "반드시 확정" 11개 항목의 권위 있는 구현 명세다. 스키마·설정·점수표·룰셋과 각 항목의 MVP 수용 기준을 포함하며, 설계 문서이자 구현·검수 계약으로 기능한다.

> 포함 섹션: §31  ·  전체 문서 맵은 `README.md` 참조. 섹션 번호는 분리 전 통합 문서와 동일하게 유지한다(상호참조 안정성).

---

## 31. MVP 구현 명세 (MVP Implementation Spec)

§30에서 확정한 MVP 결정을 구현 가능한 수준으로 명세한다. 각 항목은 스키마/설정/표와 **MVP 수용 기준(acceptance criteria)** 을 포함한다. 이 절은 §30.4 "반드시 확정" 11개 항목의 권위 있는 정본이다.

### 31.1 Shell Integration + rc 수정 UX

확정값: **Hook 기반 기본 + Native Wrapper fallback**, rc 수정은 명시적 opt-in.

MVP 필수 지원 셸: `bash`, `zsh`. 이후 지원: `fish`, `sh`, `dash`.

rc 삽입 블록(존재 시에만 평가, 미설치 시 무해):

```bash
# AI Terminal integration
if command -v ai >/dev/null 2>&1; then
  eval "$(ai shell-hook zsh)"   # bash는 'ai shell-hook bash'
fi
```

설치 UX:

```bash
ai init shell             # 설치 (확인 후)
ai init shell --dry-run   # 적용 없이 미리보기만
ai init shell --diff      # 적용 예정 diff 표시
ai init shell --uninstall # AI 터미널이 삽입한 블록만 제거
```

```diff
--- ~/.zshrc
+++ ~/.zshrc
@@ -42,3 +42,8 @@
 export PATH="$HOME/.local/bin:$PATH"
+# AI Terminal integration
+if command -v ai >/dev/null 2>&1; then
+  eval "$(ai shell-hook zsh)"
+fi
```

`--uninstall`은 삽입 블록만 제거하고 사용자 작성 라인은 절대 수정하지 않는다.

Hook 수집 이벤트:

| 이벤트 | 목적 |
|---|---|
| prompt render | 현재 cwd, exit code 동기화 |
| command preexec | 실행 명령 기록 |
| command precmd | 종료 코드·cwd·상태 갱신 |
| directory change | cwd 갱신 |
| shell startup | shell·user·hostname·env subset 초기화 |

**수용 기준**: `--dry-run`은 파일을 수정하지 않는다 / `--diff`는 수정 예정 diff를 보인다 / `--uninstall`은 삽입 블록만 제거한다 / 설치 후 `cd`·`export`·`git branch` 상태가 AI 컨텍스트에 반영된다 / Hook 실패가 일반 셸 사용을 중단시키지 않는다.

### 31.2 SQLite 스키마 + 파일 락 / Stale Lock

확정값: 데몬 없음. 단일 `ai-terminal.db`(WAL) + advisory 파일 락 + stale lock cleanup.

저장 레이아웃:

```text
~/.local/share/ai-terminal/
  ai-terminal.db   sessions/   logs/   cache/   locks/
```

PRAGMA:

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;
```

테이블 DDL:

```sql
CREATE TABLE sessions (
  id TEXT PRIMARY KEY, started_at TEXT NOT NULL, ended_at TEXT,
  shell TEXT, hostname TEXT, cwd TEXT,
  policy_profile TEXT NOT NULL, status TEXT NOT NULL
);

CREATE TABLE commands (
  id TEXT PRIMARY KEY, session_id TEXT NOT NULL,
  started_at TEXT NOT NULL, ended_at TEXT, cwd TEXT,
  command_text TEXT NOT NULL, command_hash TEXT NOT NULL,
  source TEXT NOT NULL, exit_code INTEGER,
  risk_level TEXT, risk_score INTEGER,
  confirmed INTEGER NOT NULL DEFAULT 0,
  ai_generated INTEGER NOT NULL DEFAULT 0,
  preview_id TEXT, undo_id TEXT,
  FOREIGN KEY(session_id) REFERENCES sessions(id)
);

CREATE TABLE ai_requests (
  id TEXT PRIMARY KEY, session_id TEXT NOT NULL, command_id TEXT,
  created_at TEXT NOT NULL, provider TEXT, model TEXT, intent TEXT,
  status TEXT NOT NULL, input_tokens INTEGER, output_tokens INTEGER,
  estimated_cost_usd REAL, context_hash TEXT,
  cancelled INTEGER NOT NULL DEFAULT 0, error_code TEXT,
  FOREIGN KEY(session_id) REFERENCES sessions(id),
  FOREIGN KEY(command_id) REFERENCES commands(id)
);

CREATE TABLE usage_events (
  id TEXT PRIMARY KEY, created_at TEXT NOT NULL,
  provider TEXT NOT NULL, model TEXT NOT NULL,
  input_tokens INTEGER NOT NULL DEFAULT 0,
  output_tokens INTEGER NOT NULL DEFAULT 0,
  cached_tokens INTEGER NOT NULL DEFAULT 0,
  estimated_cost_usd REAL NOT NULL DEFAULT 0,
  session_id TEXT, request_id TEXT
);

CREATE TABLE audit_events (
  id TEXT PRIMARY KEY, created_at TEXT NOT NULL,
  session_id TEXT, command_id TEXT, event_type TEXT NOT NULL,
  risk_level TEXT, policy_profile TEXT, payload_json TEXT NOT NULL
);

CREATE TABLE context_snapshots (
  id TEXT PRIMARY KEY, session_id TEXT NOT NULL, created_at TEXT NOT NULL,
  context_type TEXT NOT NULL, hostname TEXT, cwd TEXT, shell TEXT,
  git_root TEXT, git_branch TEXT, env_hash TEXT, alias_hash TEXT,
  payload_json TEXT NOT NULL,
  FOREIGN KEY(session_id) REFERENCES sessions(id)
);

CREATE TABLE locks (
  name TEXT PRIMARY KEY, owner_pid INTEGER NOT NULL, owner_session_id TEXT,
  acquired_at TEXT NOT NULL, expires_at TEXT NOT NULL, heartbeat_at TEXT NOT NULL
);
```

> 락은 **두 층**으로 둔다: `locks/` 디렉터리의 advisory 파일 락(프로세스 간 직렬화)과 `locks` 테이블(소유자·heartbeat 레지스트리, stale 판정 근거). 파일 락은 빠른 상호 배제, 테이블은 진단·복구용이다.

Lock TTL:

| Lock | TTL | 용도 |
|---|--:|---|
| db.lock | 10초 | 짧은 DB write |
| usage.lock | 10초 | usage 집계 |
| index.lock | 30분 | 인덱싱 |
| policy.lock | 10초 | 정책 갱신 |
| session.lock | 세션 생존 기간 | 세션 소유권 |

Stale 판정(다음 중 하나): PID 부재 / heartbeat가 TTL 초과 / owner session 종료 / lock 파일 mtime이 TTL 초과. 처리: ① stale 확인 → ② stale metadata를 audit log 기록 → ③ lock 제거 → ④ 재시도.

**수용 기준**: 동시 터미널 2개 이상에서 DB corruption 없음 / stale lock이 남아도 다음 실행이 복구 가능 / WAL 활성 / busy timeout 초과 시 명확한 오류 표시.

### 31.3 기본 Policy Profile (권위값)

MVP 필수: `balanced`(기본), `paranoid`.

```toml
[profiles.balanced]
confirm_level = "medium_and_above"
block_critical = true
sandbox_high_risk_commands = true
preview_file_modifications = true
show_context_scope_before_remote_call = "when_sensitive"
max_context_tokens = 32000
auto_execute = false
auto_healing = true
auto_healing_max_attempts = 1
allow_remote_ai = true
allow_sudo_ai_commands = false
mask_pii = true
mask_secrets = true
block_on_masking_failure = true
remote_approval = false

[profiles.paranoid]
confirm_level = "all_ai"
block_critical = true
sandbox_all_ai_commands = true
preview_file_modifications = true
show_context_scope_before_remote_call = true
max_context_tokens = 8000
auto_execute = false
auto_healing = false
allow_remote_ai = false
allow_sudo_ai_commands = false
mask_pii = true
mask_secrets = true
block_on_masking_failure = true
remote_approval = false
```

| 항목 | balanced | paranoid |
|---|---|---|
| AI 명령 확인 | Medium 이상 | 모든 AI 명령 |
| Critical 명령 | 차단 | 차단 |
| 원격 AI | 허용 | 차단 |
| 컨텍스트 범위 표시 | 민감할 때 | 항상 |
| Auto-healing | 제안만 | 비활성 |
| sudo AI 명령 | 금지 | 금지 |
| Preview(파일 수정) | 필수 | 필수 |
| Secret/PII 마스킹 | 필수 | 필수 |
| 원격 승인 | 비활성 | 비활성 |

**수용 기준**: `ai policy show`가 현재 정책 표시 / `ai policy set paranoid` 즉시 반영 / 두 프로파일 모두 Critical 차단 / paranoid에서 원격 AI 기본 차단.

### 31.4 위험 명령 분류 + 위험도 스코어링 (0~100, MVP 정본)

확정값: rule-based scoring이 기본, AI 판단은 보조 신호, **로컬 정책이 항상 우선**.

등급:

| 등급 | 점수 | 설명 |
|---|--:|---|
| Low | 0~24 | 읽기 중심, 부작용 없음 |
| Medium | 25~49 | 제한적 파일 변경/프로세스 제어 |
| High | 50~79 | 광범위 변경·권한 상승·네트워크 실행 |
| Critical | 80~100 | 시스템 손상·대량 삭제 가능 |

명령 유형 점수:

| 조건 | 점수 | 조건 | 점수 |
|---|--:|---|--:|
| read-only | +0 | 다운로드 후 실행 | +50 |
| 파일 생성/수정 | +20 | sudo 포함 | +40 |
| 파일 삭제 | +35 | root 경로 대상 | +50 |
| 재귀 삭제 | +30 | 홈 디렉터리 전체 | +40 |
| 권한 변경 | +25 | 시스템 디렉터리 | +50 |
| 재귀 권한 변경 | +35 | Docker socket 접근 | +60 |
| 프로세스 종료 | +25 | 디스크/파티션 조작 | +80 |
| 서비스 재시작 | +35 | 암호화/파쇄/overwrite | +70 |
| 패키지 설치/삭제 | +40 | 네트워크 다운로드 | +25 |

경로 가중치: 현재 디렉터리 내부 +0 / 프로젝트 루트 전체 +15 / `$HOME` +30 / `/etc`·`/usr`·`/bin`·`/sbin`·`/var` +50 / `/dev`·`/proc`·`/sys` +60 / `/` +60 / `/var/run/docker.sock` +70.

완화 요소: dry-run −20 / preview 완료 −10 / Git tracked만 수정 −5 / 임시 디렉터리 대상 −10 / 명시적 파일 1개 −10.

예시 분류: `ls -al`=Low / `rm ./tmp.txt`=Medium / `rm -rf ./build`=Medium~High / `rm -rf /`=Critical / `chmod -R 777 .`=High / `sudo systemctl restart nginx`=High / `curl URL | sh`=High / `dd if=/dev/zero of=/dev/sda`=Critical / `docker run --privileged`=High.

정책 액션:

| 위험도 | balanced | paranoid |
|---|---|---|
| Low | 허용 | 확인 |
| Medium | 확인 | 확인 |
| High | 강한 확인 + sandbox/preview | 기본 차단 또는 강한 확인 |
| Critical | 차단 | 차단 |

**수용 기준**: 점수가 deterministic / 동일 명령·환경은 동일 점수 / AI가 Low여도 로컬이 High면 High / Critical은 실행되지 않음.

### 31.5 Preview / Diff 적용 범위

확정값: 파일 변경 가능성 명령은 가능하면 preview/diff. native dry-run 우선, 단순 파일 수정은 임시 복사본 diff.

Preview 필수 대상: 파일 내용 변경(`sed -i`, `perl -pi`), formatter(`prettier --write`, `black`, `gofmt -w`), codemod(jscodeshift, ruff fix), 대량 rename, 삭제(`rm`, `find -delete`), 권한 변경(`chmod`, `chown`), 설정 파일 수정(.env/yaml/toml/json), AI 생성 스크립트.

Dry-run 우선 도구: `rsync --dry-run` / `git clean -n` / `terraform plan` / `kubectl --dry-run=client` / `helm template|--dry-run` / `find -delete`는 `-print` 선행 / `rm`은 대상 목록 표시.

Diff 생성: ① 대상 파일 목록 산출 → ② 임시 디렉터리 복사 → ③ 임시본에서 명령 실행 → ④ 원본 대비 diff → ⑤ 확인 후 원본 적용.

Preview 불가 조건: 외부 시스템 상태 변경, DB migration, cloud resource 변경, service restart, network transfer, interactive command, 대상 목록 산출 불가, non-deterministic 명령.

```text
Preview is not available for this command.
Reason: Command may affect external system state.
Risk: High
Run anyway? [y/N]
```

**수용 기준**: `sed -i`류는 diff preview 표시 / `rm -rf ./path`는 삭제 대상 목록·개수 표시 / preview 불가 시 이유 명확 표시 / preview 실패 시 원본 미수정.

### 31.6 Undo Backup 상한

확정값: best-effort 파일 롤백만. generic command undo 없음.

지원: 단일/제한적 다중 파일 수정, formatter·sed/perl 치환 전 백업, AI 스크립트가 수정 대상을 사전 산출한 경우. 미지원: 삭제 전체 복구 보장, DB migration, 패키지 설치/삭제, 서비스 재시작, 네트워크 전송, 클라우드 변경, Docker/k8s 변경.

```toml
[undo]
enabled = true
mode = "best_effort"
max_backup_size_mb = 500
max_file_count = 1000
max_file_size_mb = 20
backup_ttl_days = 7
exclude_binary_files = true
compress_backups = true
```

저장: `~/.local/share/ai-terminal/undo/<undo_id>/{metadata.json, files/}`.

```json
{ "undo_id": "undo_20260601_001", "created_at": "2026-06-01T10:30:00+09:00",
  "command_id": "cmd_123", "mode": "best_effort", "file_count": 3,
  "backup_size_bytes": 42000, "expires_at": "2026-06-08T10:30:00+09:00" }
```

**수용 기준**: 백업 한도 초과 시 실행 전 사용자 고지 / 백업 실패 시 위험 명령 중단 / `ai undo last`가 지원 대상 복구 / undo 불가 명령은 이유 표시.

### 31.7 Token / Cost Usage Schema

확정값: 모든 AI 요청에 usage event 기록, 실제 사용량 미제공 시 estimated 표시.

```json
{ "id": "usage_123", "created_at": "2026-06-01T10:30:00+09:00",
  "session_id": "sess_123", "request_id": "req_123",
  "provider": "example-provider", "model": "example-model",
  "input_tokens": 3240, "output_tokens": 620, "cached_tokens": 0,
  "token_count_source": "provider_reported",
  "estimated_cost_usd": 0.0041, "cost_source": "estimated",
  "currency": "USD", "budget_scope": "session" }
```

`token_count_source`: `provider_reported` / `local_tokenizer` / `estimated` / `unknown`.
`cost_source`: `provider_reported` / `pricing_table` / `estimated` / `unknown`.

```toml
[ai.usage]
show_token_usage = true
show_estimated_cost = true
session_budget_usd = 2.00
monthly_budget_usd = 30.00
warn_at_budget_percent = 80
block_at_budget_percent = 100
```

**수용 기준**: 모든 AI 요청이 usage event 기록 / 부정확 비용은 estimated 표시 / 예산 초과 시 원격 AI 차단 가능 / 로컬 LLM은 비용 0 또는 local resource usage로 표시.

### 31.8 Secret / PII Masking Baseline

확정값: `mask_secrets=true`, `mask_pii=true`, `block_remote_ai_on_masking_failure=true`.

Secret 필수 탐지: API key, Bearer/OAuth/Refresh token, Password, SSH private key, Private key block, Database URL, AWS access key, GitHub/Slack token, Cookie, Authorization header, Session ID.

PII 필수 탐지: 이메일, IPv4, 전화번호, 한국 주민등록번호 패턴, 여권번호 유사, 신용카드 유사, 위치성 주소 일부.

```yaml
masking_rules:
  - name: email
    type: pii
    pattern: '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}'
    replacement: '[EMAIL_REDACTED]'
    enabled: true
  - name: ipv4
    type: pii
    pattern: '\b(?:\d{1,3}\.){3}\d{1,3}\b'
    replacement: '[IP_REDACTED]'
    enabled: true
  - name: bearer_token
    type: secret
    pattern: 'Bearer\s+[A-Za-z0-9._~+/=-]+'
    replacement: 'Bearer [TOKEN_REDACTED]'
    enabled: true
  - name: aws_access_key
    type: secret
    pattern: 'AKIA[0-9A-Z]{16}'
    replacement: '[AWS_ACCESS_KEY_REDACTED]'
    enabled: true
  - name: private_key_block
    type: secret
    pattern: '-----BEGIN [A-Z ]*PRIVATE KEY-----'
    replacement: '[PRIVATE_KEY_REDACTED]'
    enabled: true
  - name: kr_rrn
    type: pii
    pattern: '\b\d{6}-?[1-4]\d{6}\b'
    replacement: '[KR_RRN_REDACTED]'
    enabled: true
```

순서: Raw Context → Secret Detection → PII Detection → Masking → Validation Scan → Remote AI Eligibility Check.

**수용 기준**: `.env`는 기본적으로 원격 컨텍스트 제외 / private key block 감지 시 원격 호출 차단 / 마스킹된 값만 세션 로그 저장 / 원문 secret은 디스크 미저장.

### 31.9 Provider Capability Map Schema

확정값: provider 차이를 숨기지 않고 capability map으로 관리. 완전 추상화는 비목표.

```json
{ "provider": "example", "display_name": "Example AI",
  "models": [{
    "name": "example-model", "max_context_tokens": 128000, "max_output_tokens": 4096,
    "supports_streaming": true, "supports_json_mode": true, "supports_tool_use": false,
    "supports_token_counting": false, "supports_usage_reporting": true,
    "supports_context_caching": false, "supports_system_prompt": true, "supports_seed": false,
    "token_counting_method": "estimated", "usage_reporting_method": "response_metadata",
    "cost_tracking_method": "pricing_table", "default_timeout_seconds": 60 }] }
```

필수 capability: `supports_streaming`, `supports_json_mode`, `supports_tool_use`, `supports_token_counting`, `supports_usage_reporting`, `supports_context_caching`, `max_context_tokens`, `max_output_tokens`, `token_counting_method`, `cost_tracking_method`.

Provider 정책: streaming 미지원 → non-streaming UI fallback / token counting 미지원 → estimated / usage reporting 미지원 → pricing table 추정 / JSON mode 미지원 → 응답 파서 보수화 / tool use는 MVP 핵심 제외.

**수용 기준**: capability가 설정/registry로 로드 / 미지원 기능은 명시적 fallback / 불확실한 비용·토큰은 estimated / provider 변경 시 정책 엔진 동일 적용.

### 31.10 Context Sync Hook 범위

확정값: 정확성보다 안정성 우선. 전체 shell state를 복제하지 않고 핵심 상태만 추적.

필수 추적: `cwd`, `last_command`, `last_exit_code`, `shell`, `hostname`, `user`, `git_root`, `git_branch`, `git_dirty_state`, `active_policy_profile`, `ai_mode`.
선택 추적: 선택 환경 변수, alias hash, PATH hash, virtualenv/conda/node/python 버전, docker/kube context.

상태 갱신 트리거 built-in: `cd`, `pushd`, `popd`, `export`, `unset`, `alias`, `unalias`, `source`, `.`, `direnv allow`, `conda activate/deactivate`, `pyenv shell`, `nvm use`, `git checkout`, `git switch`, `git pull`, `git reset`.

Env 수집은 전체 저장 금지, allowlist 기반:

```toml
[context.env]
allowlist = ["PATH","SHELL","USER","HOME","PWD","VIRTUAL_ENV","CONDA_DEFAULT_ENV","NODE_ENV"]
hash_only = ["PATH"]
denylist_patterns = [".*TOKEN.*",".*SECRET.*",".*KEY.*",".*PASSWORD.*"]
```

Context refresh 트리거: AI context cwd ≠ 실제 `pwd` / git branch 변경 / SSH·container·k8s context 변경 / hook heartbeat 끊김 / 마지막 snapshot이 5분 이상 경과.

**수용 기준**: `cd` 이후 새 cwd 인식 / `git switch` 이후 branch 갱신 / env 내 secret 미저장 / mismatch 감지 시 AI 명령 생성 전 refresh 제안.

### 31.11 플랫폼별 Guardrails Capability Matrix

확정값: 모든 플랫폼 동일 동적 guardrails를 보장하지 않음. baseline과 platform-specific을 구분.

Baseline(모든 플랫폼): static command analysis, risk scoring, preview/diff(가능 시), dry-run(가능 시), timeout, process group termination, confirmation prompt, secret/PII masking, policy profile enforcement.

| 기능 | Linux | WSL | macOS |
|---|---|---|---|
| Static risk analysis / Preview/diff / Timeout | 지원 | 지원 | 지원 |
| Process group termination | 지원 | 부분 | 지원 |
| File count pre-scan | 지원 | 지원 | 지원 |
| cgroups CPU/mem limit | 지원 | 부분 | 미지원 |
| seccomp / fanotify | 지원 | 제한 | 미지원 |
| inotify | 지원 | 지원 | 미지원 |
| FSEvents | 미지원 | 미지원 | 지원 가능 |
| Docker sandbox | 지원 | Docker Desktop 의존 | Docker Desktop 의존 |
| bubblewrap | 지원 | 제한 | 미지원 |
| gVisor / eBPF | Phase 3 | 제한 | 별도 구현 |

Linux MVP 우선 구현: static risk analysis, preview/diff, timeout, process group kill, file count pre-scan, Docker socket 차단, 주요 시스템 경로 차단, sandbox resource limit.

WSL/macOS는 제한 사항을 `ai doctor --guardrails`로 명시 고지하고, 동적 감시가 제한되는 플랫폼에서는 High 이상 명령 확인을 강화한다.

**수용 기준**: `ai doctor --guardrails`가 현재 플랫폼 capability 표시 / 미지원 guardrail은 조용히 실패하지 않고 명시 / 모든 플랫폼에서 static analysis·preview/diff 동작 / 동적 감시 제한 플랫폼은 High+ 확인 강화.

### 31.12 MVP 진입 승인 체크리스트

각 영역이 완료되어야 구현에 진입한다. (요약 — 세부는 §31.1~31.11)

- **Shell**: Hook 통합·bash/zsh 범위·rc dry-run/diff/uninstall·Wrapper fallback
- **Storage**: WAL·sessions/commands/ai_requests/usage_events/audit_events/context_snapshots 스키마·락 TTL·stale 복구
- **Policy & Risk**: balanced/paranoid·위험도 점수표·Critical block·sudo 정책·remote AI 정책
- **Preview & Undo**: preview 대상·diff 방식·dry-run 도구·undo 범위·백업 상한·불가 UX
- **Usage**: usage event schema·token/cost source·budget 동작·estimated 정책
- **Privacy**: Secret/PII baseline·`.env` 처리·private key 차단·마스킹 실패 차단
- **Provider**: capability map schema·streaming/token/usage/JSON fallback
- **Context Sync**: 필수 field·hook built-in·env allow/deny·mismatch 기준·snapshot TTL
- **Guardrails**: baseline·Linux/WSL/macOS capability·`ai doctor --guardrails` 출력

### 31.13 최종 MVP 진입 결정 (확정값)

```text
Shell    : Hook default + Native Wrapper fallback; rc dry-run/diff/uninstall required
Storage  : SQLite WAL + file lock + stale lock cleanup (ai-terminal.db)
Policy   : balanced default + paranoid required; local policy wins
Risk     : rule-based 0~100 score; Critical blocked; AI classification advisory only
Preview  : required for feasible file modification; dry-run first; diff via temp copy
Undo     : best-effort file rollback only; 500MB cap; 7-day TTL
Usage    : usage event per AI request; estimated when provider data unavailable
Privacy  : Secret/PII masking on by default; block remote AI on masking failure
Provider : minimal interface + capability map required
Context  : cwd/exit code/git state/shell/hostname required; env allowlist only
Guardrails: static analysis + preview + timeout baseline; platform capability matrix
```

---
