# 보안 · 프라이버시 · 정책

보안/프라이버시 설계와 정책 체계를 정의한다: 위험 명령 분류, Zero-Trust Context Pipeline, Secret/PII 마스킹, 감사 로그, 정책 엔진·정책 프로파일·조직 정책.

> 포함 섹션: §10, §12  ·  전체 문서 맵은 `README.md` 참조. 섹션 번호는 분리 전 통합 문서와 동일하게 유지한다(상호참조 안정성).

---

## 10. 보안 및 프라이버시 설계

### 10.1 위험 명령어 분류

| 등급 | 설명 | 예시 | 매핑 권한(§17) |
|---|---|---|---|
| Low | 읽기 전용/영향 작음 | `ls`, `cat`, `pwd` | ReadOnly |
| Medium | 파일 변경/프로세스 제어 | `rm ./file`, `kill`, `chmod` | FileWrite / ProcessControl |
| High | 광범위 삭제·권한 변경·네트워크 전송 | `rm -rf`, `curl \| sh`, `scp` | Network / Privileged |
| Critical | 시스템 손상 가능성 높음 | `mkfs`, `dd`, `chmod -R 777 /` | Destructive |

### 10.2 실행 전 확인 정책

다음에 해당하면 실행 전 확인을 요구한다: `sudo` 포함, `rm -rf` 포함, 루트 경로 대상, 홈 디렉터리 전체 대상, 대량 파일 삭제, 권한 재귀 변경, 디스크 포맷, 원격 코드 다운로드 후 실행, 네트워크 파일 전송, 환경 변수/시크릿 출력 가능성.

### 10.3 Zero-Trust Context Pipeline

AI에 전달되는 모든 컨텍스트(로그·README·주석·에러 메시지·외부 파일)는 신뢰할 수 없는 데이터로 취급한다. 원격 AI 호출 전 다음을 반드시 통과한다.

```text
Raw Context Collection → Secret/PII Detection → Masking → Data Minimization
   → Prompt Injection Scan → Local Policy Evaluation → Consent & Scope Display
   → Remote AI Call or Local AI Call
```

- **Data Minimization**: 전체 로그 대신 에러 주변 N라인, 전체 `.env` 대신 키 이름만, 전체 트리 대신 관련 디렉터리만.
- **Local Policy Evaluation(로컬 우선)**: Secret/PII 포함 여부, 원격 호출 허용 여부, 위험도, 가용 컨텍스트 범위, 조직 정책 위반, 비용 예산, 샌드박스 필요 여부.
- **Consent & Scope Display**:

```text
The following context will be sent to the remote AI provider:
- User request / Current directory path / Last command and exit code
- 120 lines of stderr / Git branch name / package.json scripts
Masked: 3 email addresses / 1 API token / 2 IP addresses
Send? [Y/n]
```

`paranoid` 프로파일에서는 이 확인을 항상 요구한다.

### 10.4 Secret 및 PII 마스킹

AI 요청·세션 로그·감사 로그·오류 분석 컨텍스트는 AI Service Layer 전달 전 마스킹 파이프라인을 반드시 통과한다.

**Secret**: API Key, Access/Refresh Token, Password, Private/SSH Key, OAuth Credential, Database URL, Cloud Credential, Cookie, Authorization Header, Session ID.

**PII**: 이메일, 전화번호, IP 주소, 주민등록번호/국가 식별번호, 여권번호, 주소, 이름+조직 결합 항목, 사용자 ID, 결제 정보, 위치 정보.

정규식 기반 예시:

```yaml
masking_rules:
  - name: email
    pattern: '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}'
    replacement: '[EMAIL_REDACTED]'
  - name: ipv4
    pattern: '\b(?:\d{1,3}\.){3}\d{1,3}\b'
    replacement: '[IP_REDACTED]'
  - name: phone_kr
    pattern: '\b01[016789]-?\d{3,4}-?\d{4}\b'
    replacement: '[PHONE_REDACTED]'
  - name: resident_registration_number_kr
    pattern: '\b\d{6}-?[1-4]\d{6}\b'
    replacement: '[KR_RRN_REDACTED]'
  - name: bearer_token
    pattern: 'Bearer\s+[A-Za-z0-9._~+/=-]+'
    replacement: 'Bearer [TOKEN_REDACTED]'
  - name: aws_access_key
    pattern: 'AKIA[0-9A-Z]{16}'
    replacement: '[AWS_ACCESS_KEY_REDACTED]'
  - name: private_key
    pattern: '-----BEGIN [A-Z ]*PRIVATE KEY-----'
    replacement: '[PRIVATE_KEY_REDACTED]'
```

마스킹 정책:

- AI 요청 전·세션 로그 저장 전·감사 로그 저장 전 모두 마스킹 적용.
- 사용자가 명시 허용하지 않는 한 `.env`·private key·credential 파일은 컨텍스트에 포함하지 않는다.
- 마스킹 전 원문은 메모리에서만 일시 사용하고 디스크에 저장하지 않는다.
- **마스킹 실패 감지 시 원격 AI 호출 중단**:

```text
Sensitive data may be present in the selected context.
Remote AI request blocked by privacy policy.
Use local AI mode or reduce context scope.
```

> 정규식 기반 마스킹은 완전하지 않다(§29.8 참고: 엔트로피 기반 시크릿 탐지, 거부 시 fail-closed).

### 10.5 감사 로그 + Before/After Snapshot

중요 명령은 감사 로그에 기록한다.

```json
{ "timestamp": "2026-06-01T10:30:00+09:00", "user": "local-user", "cwd": "/home/user/app",
  "command": "rm -rf ./build", "source": "ai-generated", "risk_level": "medium",
  "confirmed": true, "exit_code": 0 }
```

대상 명령(파일 삭제/권한 변경, 대량 rename, 설정 수정, 패키지 설치/삭제, 서비스 재시작, 배포, DB 마이그레이션, 인프라 변경)은 실행 전후 상태 스냅샷을 선택적으로 기록한다.

```json
{
  "before": { "cwd": "/home/user/app", "git_branch": "main", "git_status": "modified",
              "affected_files": ["src/app.ts"], "disk_usage": "12G" },
  "after":  { "exit_code": 0, "git_diff_stat": "1 file changed, 3 insertions(+), 1 deletion(-)",
              "affected_files": ["src/app.ts"] }
}
```

Git 관리 디렉터리에서는 실행 전후 `git diff --stat` / `git diff --name-only`를 자동 기록한다. **원문 diff 전체는 민감 정보 가능성이 있으므로 기본 저장하지 않는다.**

---

## 12. 정책 엔진, 정책 프로파일, 조직 정책

### 12.1 Policy Engine (정적)

명령 실행 전 패턴 매칭 기반으로 allow/confirm/block을 결정한다.

```yaml
rules:
  - match: "rm -rf /"
    action: "block"
    reason: "Root filesystem deletion attempt"
  - match: "curl * | sh"
    action: "confirm"
    risk: "high"
    reason: "Remote script execution"
  - match: "sudo *"
    action: "confirm"
    risk: "high"
    reason: "Privileged command"
  - match: "ls *"
    action: "allow"
    risk: "low"
```

기본 정책: AI 생성 명령은 자동 실행하지 않음 / 위험 명령은 설명·확인 / Critical은 기본 차단 / 정책 완화는 가능하되 경고 표시 / 모든 AI 생성 명령은 히스토리에 별도 표시.

### 12.2 Policy Profile

사용자·환경에 따라 보안 수준을 전환한다. **MVP 필수는 `balanced`(기본)와 `paranoid` 둘이며, 그 권위 있는 전체 필드값은 §31.3에 정의한다.** `poweruser`·`dev`는 Phase 2 확장 프로파일이다.

| 프로파일 | 설명 |
|---|---|
| `paranoid` | 모든 AI 명령 확인, 원격 AI 제한, 강한 마스킹 |
| `balanced` | 기본 권장, 위험 명령 확인, 일반 명령 허용 |
| `poweruser` | 숙련자용, 확인 절차 최소화 |
| `dev` | 개발용, preview·자동 복구 중심 |

```toml
[policies]
default_profile = "balanced"
available_profiles = ["paranoid", "balanced", "poweruser", "dev"]

[profiles.paranoid]
confirm_level = "all_ai"
sandbox_all_ai_commands = true
block_remote_ai = true
show_context_scope_before_remote_call = true
max_context_tokens = 8000
auto_execute = false
auto_healing = false
mask_pii = true
mask_secrets = true
block_on_masking_failure = true

[profiles.balanced]
confirm_level = "medium_and_above"
sandbox_high_risk_commands = true
block_remote_ai = false
show_context_scope_before_remote_call = "when_sensitive"
max_context_tokens = 32000
auto_execute = false
auto_healing = true
mask_pii = true
mask_secrets = true

[profiles.poweruser]
confirm_level = "high_and_above"
sandbox_high_risk_commands = false
block_remote_ai = false
show_context_scope_before_remote_call = false
max_context_tokens = 64000
auto_execute = false
auto_healing = true

[profiles.dev]
confirm_level = "medium_and_above"
sandbox_high_risk_commands = true
preview_file_modifications = true
auto_healing = true
max_context_tokens = 32000
```

전환 명령: `ai policy show` / `ai policy set <profile>`.

### 12.3 조직 정책 (Phase 3)

정책 파일 우선순위:

```text
Organization Policy > Team Policy > User Policy > Default Policy
```

```text
~/.config/ai-terminal/policy.d/
  00-org-policy.toml
  10-team-policy.toml
  20-user-policy.toml
```

조직 정책은 readonly로 적용될 수 있으며 사용자가 완화할 수 없다.

```toml
[organization]
name = "Example Corp"
readonly = true

[ai]
block_remote_ai = true
allowed_providers = ["local"]
mask_pii = true
mask_secrets = true

[commands]
block_patterns = ["curl * | sh", "chmod -R 777 /", "rm -rf /", "docker run --privileged *"]

[audit]
required = true
log_path = "/var/log/ai-terminal/audit.jsonl"
```

---
