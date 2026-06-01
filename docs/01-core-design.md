# 핵심 컴포넌트 · 런타임 설계

터미널의 핵심 컴포넌트와 런타임 동작을 정의한다: 5계층 컴포넌트 상세, Context Consistency Manager, Execution Guardrails Engine, AI CLI 명령어, 세션 컨텍스트, 권한 모델, 일반 터미널 호환성, 엣지 케이스.

> 포함 섹션: §6, §7, §8, §9, §11, §17, §18, §19  ·  전체 문서 맵은 `README.md` 참조. 섹션 번호는 분리 전 통합 문서와 동일하게 유지한다(상호참조 안정성).

---

## 6. 컴포넌트 상세 설계

### 6.1 Terminal UI Layer

사용자 입력과 출력 표시를 담당한다.

#### 책임

- 키 입력 처리, 명령어 자동완성, 프롬프트 표시
- stdout/stderr 렌더링, AI 추천 결과 표시
- 명령 실행 전 확인 UI, 멀티라인 입력, 복사/붙여넣기 보안 경고
- AI 요청 인터럽트 처리, 스트리밍 제안 표시

#### UI 모드

| 모드 | 호출 | 설명 |
|---|---|---|
| Shell Mode | (기본) | 일반 리눅스 터미널 |
| AI Inline Mode | `ai "..."` | 쉘 프롬프트에서 접두어로 AI 호출 |
| AI Dedicated Mode | `ai shell` | 대화형 전용 모드 |
| Hybrid Mode | `ai shell` 내부 | AI 대화 + `!cmd` 즉시 실행 혼합 |

##### Hybrid Mode 입력 규칙

```bash
$ ai shell
ai> nginx 500 에러 원인 분석해줘      # 자연어 → AI 요청
ai> !tail -n 100 /var/log/nginx/error.log  # 일반 쉘 명령 즉시 실행
ai> 이 로그 기준으로 원인 요약해줘
ai> /status                          # AI shell 내부 제어 명령
ai> /exit
```

| 입력 | 동작 |
|---|---|
| 일반 자연어 | AI 요청으로 처리 |
| `!command` | 일반 쉘 명령으로 즉시 실행 |
| `/command` | AI shell 내부 제어 명령 |
| `Ctrl+C` | 현재 AI 요청 또는 실행 중 명령 취소 |
| `Ctrl+D` | AI shell 종료 |

내부 제어 명령: `/status`(컨텍스트 표시), `/context`(전송 범위 표시), `/shell`(쉘 모드 전환), `/clear`(대화 초기화), `/policy`(정책 표시), `/usage`(토큰·비용), `/exit`.

#### AI 요청 인터럽트 및 복구 (Ctrl+C)

AI 요청은 일반 쉘 명령 실행과 분리된 **취소 가능한 비동기 작업 단위**로 관리한다. 응답 대기 중 `Ctrl+C`는 터미널 프로세스를 종료하지 않고 현재 AI 요청만 취소한 뒤 즉시 쉘 프롬프트로 복귀한다.

```text
User runs AI command
   |  AI request started asynchronously
   +--> User presses Ctrl+C
           |  Cancel AI request → Release UI lock → Return to shell prompt
```

요구사항:

- `Ctrl+C`는 AI 요청 취소와 쉘 프로세스 인터럽트를 구분 처리한다.
- 취소 후 PTY 상태가 손상되지 않아야 한다.
- 스트리밍 중 취소해도 출력 버퍼가 깨지지 않아야 한다.
- 네트워크 장애·응답 지연·타임아웃에도 터미널은 프리징되지 않아야 한다.
- 취소된 요청은 세션 로그에 `cancelled` 상태로 기록한다.

```json
{ "type": "ai_request", "status": "cancelled", "reason": "user_interrupt", "timestamp": "2026-06-01T11:20:00+09:00" }
```

#### Visual Command Preview Pane

AI가 명령어/파일 변경을 제안할 때 별도 패널에 실행 전 정보를 시각적으로 표시한다.

표시 항목: 생성된 명령어, 위험도, 예상 변경 파일 목록, diff preview, dry-run 결과, 토큰 사용량, 예상 비용, 실행 컨텍스트, 정책 프로파일, 요구 권한.

```text
+---------------- Terminal ----------------+---------------- Preview ----------------+
| $ ai run "console.log 제거해줘"           | Command: sed -i '/console.log/d' app.js |
|                                          | Risk: Medium   Files affected: 1        |
|                                          | Diff: - console.log("debug")            |
|                                          | Apply? [y/N]                            |
+------------------------------------------+-----------------------------------------+
```

#### 실시간 스트리밍 제안

긴 명령어/스크립트 생성 중 중간 결과를 스트리밍한다. **스트리밍 중인 명령어는 완성 전까지 실행할 수 없다.**

```text
Generating command...
[partial] find . -type f -name "*.log" ...
---
Generated command:
find . -type f -name "*.log" -mtime +7 -print
Risk: Low   Run? [y/N]
```

---

### 6.2 Command Orchestration Layer

입력을 분석해 적절한 실행 경로로 전달한다.

#### 입력 분류

| 유형 | 예시 | 처리 방식 |
|---|---|---|
| 일반 쉘 명령 | `ls -al` | Shell Executor |
| AI 명령 | `ai "..."` | AI Service |
| 내부 명령 | `:history`, `:policy` | Internal Handler |
| 위험 명령 | `rm -rf`, `dd`, `mkfs` | Policy Engine 검사 |
| 자동화 명령 | `ai runbook create` | AI Planner + Script Generator |

#### 명령 처리 흐름

```text
User Input → Input Parser → Command Classifier
  +--> Shell Command  → Policy Check → PTY Execution → (Guardrails 동적 감시)
  +--> AI Command     → Context 동기화 → Zero-Trust Pipeline → AI 응답 → Suggest/Explain/Preview/Execute
  +--> Internal       → Internal Handler
```

이 계층은 다음 두 신규 컴포넌트를 포함한다: **Policy Engine**(정적, §12), **Execution Guardrails Engine**(동적, §8), **Context Consistency Manager**(§7).

---

### 6.3 Execution Layer

#### PTY Manager

- pseudo-terminal 생성, 쉘 프로세스 입출력 연결
- 인터랙티브 명령 지원, 터미널 크기 변경 이벤트 처리

#### Shell Process

지원 대상: `bash`, `zsh`, `fish`, `sh`, `dash`. 기본 쉘은 사용자 설정에 따른다.

#### Sandbox Executor (보안 강화)

AI 생성 명령 또는 위험 가능성 명령은 샌드박스에서 사전 실행할 수 있다. **컨테이너 자체를 완전한 보안 경계로 가정하지 않으며 최소 권한 원칙을 적용한다.**

필수 보안 옵션:

- root 사용자 실행 금지, User Namespace 활성화
- 불필요한 Linux capabilities 제거 (`cap_drop: ALL` 후 최소 부여)
- privileged 컨테이너 금지, `no-new-privileges` 적용
- 호스트 파일시스템 read-write 마운트 금지, Docker socket 마운트 금지
- host network / host PID namespace 금지
- seccomp + AppArmor/SELinux 프로파일 적용
- 네트워크 차단/제한, 임시 파일시스템 사용
- CPU/메모리/PID 수 제한

```yaml
sandbox:
  container:
    privileged: false
    user: "1000:1000"
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    network: "none"
    read_only_rootfs: true
    pids_limit: 128
    memory_limit: "512m"
    cpu_limit: "1.0"
```

금지 마운트(기본): `/var/run/docker.sock`, `/`, `/etc`, `/home`, `/root`, `/proc`, `/sys`, `/dev`. 필요 시에도 읽기 전용 또는 임시 복사본 사용.

> 샌드박스 한계 고지: 샌드박스 실행 성공이 실제 호스트 환경의 안전을 완전히 보장하지는 않는다.
> `Sandbox completed successfully, but host execution may still have side effects. Review the command before running.`

샌드박스 실행이 실패하면 실제 실행으로 **자동 전환하지 않는다.**

```text
Sandbox execution failed. Reason: container runtime unavailable.
Run directly without sandbox? [y/N]
```

#### Remote Executor

지원 방식: SSH, 컨테이너 exec, Kubernetes exec, WSL, Dev Container. 원격 실행 시 컨텍스트 스택(§11)으로 로컬/원격을 구분한다.

---

### 6.4 AI Service Layer

자연어 요청을 처리해 명령어·설명·분석 결과를 생성한다.

#### 주요 기능

자연어→명령어 변환, 명령어 의미 설명, 위험도 분석, 오류 원인 분석, 로그 요약, 스크립트 생성, 자동화 플로우 제안, 프로젝트별 컨텍스트 반영.

#### AI 요청 컨텍스트 (마스킹 후)

```json
{
  "user_request": "현재 디렉터리에서 오래된 로그 파일 삭제 명령어 만들어줘",
  "current_directory": "/home/user/app",
  "shell": "bash", "os": "linux",
  "last_command": "ls logs", "last_exit_code": 0,
  "recent_stdout": "...", "recent_stderr": "...",
  "git_branch": "main",
  "project_files": ["package.json", "Dockerfile", "README.md"],
  "policy_level": "safe"
}
```

#### 6.4.1 Token Window Management

대용량 로그·긴 히스토리·큰 파일·전체 프로젝트 구조를 그대로 전달하지 않는다. 모델 호출 전 컨텍스트 정제 파이프라인을 둔다.

```text
Raw Context → Secret/PII Masking → Classification → Chunking
   → Relevance Scoring → Compression/Summarization → Token Budget → Prompt Assembly
```

컨텍스트 유형별 전략:

| 유형 | 처리 방식 |
|---|---|
| 최근 명령 히스토리 | 최근 N개 우선, 실패 명령 가중치 부여 |
| stdout/stderr | 마지막 부분 우선, 에러 키워드 중심 추출 |
| 대용량 로그 | 청크 분할 후 단계적 요약 |
| 프로젝트 파일 목록 | 중요 파일 우선 |
| 설정 파일 | 마스킹 후 일부 |
| Git diff | 변경 파일·관련 라인 중심 |
| 패키지 파일 | `package.json`, `pyproject.toml`, `Cargo.toml` 등 우선 |

청크 기준: 기본 8,000~16,000 tokens, overlap 5~10%, 에러 로그는 timestamp/severity/stack trace 기준 분할, JSON 로그는 record 단위, multiline stack trace는 단일 청크 보존.

슬라이딩 윈도우(실시간 로그·빌드·배포·테스트 실패 분석):

```text
[Window 1] lines 1-500 / [Window 2] lines 401-900 / [Window 3] lines 801-1300
```

토큰 예산 배분:

| 목적 | 시스템 | 사용자 | 컨텍스트 | 응답 |
|---|--:|--:|--:|--:|
| 명령어 생성 | 15% | 10% | 45% | 30% |
| 에러 분석 | 10% | 10% | 60% | 20% |
| 로그 요약 | 10% | 5% | 70% | 15% |
| 스크립트 생성 | 15% | 15% | 40% | 30% |

한도 초과 시 낮은 우선순위 컨텍스트부터 제거한다.

#### 6.4.2 Agent Architecture (Phase 2)

단순 명령 생성을 넘어 프로젝트 분석·오류 진단·자동화로 확장될 때 내부 agent pipeline을 적용한다.

```text
User Intent → Intent Classifier → Tool Use Planner → Parallel Tool Executor
   → Context Retriever → Command Generator → Verification Agent → Response Formatter
```

**Intent Classifier** 유형: `explain`, `fix`, `create`, `analyze`, `preview`, `search`, `operate`.

**Tool Use Planner** 예시(배포 실패 분석): ① last command/exit code 읽기 ② stderr 수집 ③ git branch 확인 ④ 패키지 매니저 확인 ⑤ 배포 스크립트 확인 ⑥ 원인 요약.

**Parallel Tool Executor**: `git status`, `git diff --stat`, `docker ps`, `systemctl status`, 로그 tail, 설정 파일 읽기, 포트 리스닝 확인 등을 병렬 수행하되 정책 엔진·권한 검사를 통과해야 한다.

**Verification Agent**: 생성 명령을 별도로 재검토.

```json
{ "verified": true, "risk_level": "medium", "requires_preview": true,
  "requires_confirmation": true, "reason": "The command modifies files in src directory." }
```

검증 항목: 요청-명령 목적 일치, 실행 컨텍스트 일치, 위험 명령 포함 여부, shell escaping 적절성, 예상 부작용, 더 안전한 dry-run 대안 존재 여부, preview 가능 여부.

#### 6.4.3 Context Retriever 고도화 (Phase 2)

- **Semantic File Index**: SQLite FTS, sqlite-vss, LanceDB, Tantivy, ripgrep, tree-sitter 기반 코드 구조 인덱싱.
- **Project Knowledge Graph**: package manager 설정, Dockerfile, compose, k8s manifest, Terraform, CI/CD, Makefile, README, entrypoint, 테스트 설정, env 템플릿 구조화.

```json
{ "project_type": "node", "entrypoints": ["src/index.ts"], "package_manager": "pnpm",
  "scripts": { "dev": "vite", "build": "tsc && vite build", "deploy": "./scripts/deploy.sh" },
  "docker": { "dockerfile": "Dockerfile", "compose": "docker-compose.yml" } }
```

Context Retrieval 우선순위: ① 요청 직접 관련 파일 ② 직전 실패 명령 stderr/stdout ③ 현재 Git diff ④ 프로젝트 설정 파일 ⑤ 최근 히스토리 ⑥ 전체 파일 트리 요약 ⑦ 오래된 세션 정보.

---

### 6.5 Storage & Audit Layer

§15(데이터 저장)와 §10.5(감사 로그)에서 상세히 다룬다. 세션 로그(jsonl), 명령 히스토리(SQLite), 정책/감사 로그, 사용량 로그, 사용자 설정/정책 프로파일을 관리한다.

---

## 7. Context Consistency Manager

### 7.1 개요

실제 PTY/Shell 상태와 AI가 인식하는 컨텍스트 간 불일치를 방지하는 컴포넌트이다. AI 터미널에서 가장 흔한 위험은 AI가 현재 cwd·환경 변수·원격 접속 상태·컨테이너 내부 여부·Git 브랜치를 잘못 이해한 채 명령을 제안하는 것이다. 쉘 상태가 바뀌는 명령이 실행될 때마다 AI 컨텍스트를 동기화한다.

### 7.2 추적 상태

현재 cwd, 쉘 종류, 환경 변수, 쉘 변수, alias/함수 정의, Git repo 상태·branch, SSH 접속 여부, 컨테이너 내부 여부, Kubernetes context, 언어 런타임(virtualenv/Node/Python) 상태, 마지막 명령·종료 코드, 마지막 stdout/stderr 요약, sudo 권한 캐시 여부.

### 7.3 Built-in 명령 동기화

다음 명령은 외부 프로세스 실행만으로는 쉘 상태 변경을 파악하기 어렵다.

```text
cd  export  unset  alias  unalias  source  .  set  umask
pushd  popd  direnv allow  conda activate  pyenv shell  nvm use
```

실행 후 동기화 훅 호출:

```text
Built-in command executed → Shell state probe → Context diff 계산 → AI context update → Session log update
```

### 7.4 Shell State Probe

```bash
pwd / env / alias / set / git rev-parse --show-toplevel / git branch --show-current / hostname / whoami
```

`env`, `set` 출력에는 민감 정보가 포함될 수 있으므로 Secret/PII 마스킹을 먼저 적용한다.

### 7.5 Context Stack

실행 위치가 바뀌면 컨텍스트를 stack으로 관리한다.

```json
[
  { "type": "local", "hostname": "laptop", "cwd": "/home/user/project" },
  { "type": "ssh", "hostname": "prod-web-01", "cwd": "/var/www/app" },
  { "type": "container", "name": "app", "cwd": "/usr/src/app" }
]
```

프롬프트는 현재 실행 컨텍스트를 명확히 표시한다.

```text
[local:/home/user/project]$
[ssh:prod-web-01:/var/www/app]$
[container:app:/usr/src/app]$
[k8s:prod/default/api-pod:/app]$
```

### 7.6 컨텍스트 불일치 감지

```text
Context mismatch detected.
AI context cwd:   /home/user/project
Actual shell cwd: /home/user/project/backend
Refresh AI context before generating command? [Y/n]
```

---

## 8. Execution Guardrails Engine

### 8.1 개요

Policy Engine(실행 전 정적 분석)과 달리, Guardrails Engine은 **실행 중 동적 감시**를 담당한다. 정적 검사를 통과한 명령이 실행 도중 비정상 행위를 보일 때 이를 차단한다.

### 8.2 감시 대상

대량 파일 삭제/권한 변경, 루트 디렉터리 접근, 홈 디렉터리 전체 변경, 비정상 네트워크 연결 증가, 예상보다 긴 실행 시간, 과도한 CPU/메모리, 프로세스 폭증, 디스크 급증, 민감 파일 접근, SSH key/credential 접근, Docker socket 접근, `/etc`·`/proc`·`/sys`·`/dev` 접근.

### 8.3 실행 중 차단 흐름

```text
Command started → Runtime monitor attached → Behavior classification
  +--> Normal     → Continue
  +--> Suspicious → Pause or ask user
  +--> Dangerous  → Terminate process group → Show incident summary
```

```text
Execution paused. The command is attempting to delete more than 1,000 files.
Command: rm -rf ./cache   Affected path: ./cache   Continue? [y/N]
---
Command terminated by guardrails.
Reason: Attempted to access /var/run/docker.sock; current policy profile blocks Docker socket access.
```

### 8.4 구현 고려사항

Linux 동적 감시 후보: cgroups(CPU/메모리/PID 제한), inotify/fanotify(파일 이벤트), seccomp(syscall 제한), eBPF(고급 이벤트), network namespace(네트워크 제한), AppArmor/SELinux(접근 제한).

**MVP 우선 구현**: 실행 시간 제한, 프로세스 그룹 단위 종료, 삭제 대상 파일 수 사전 계산, 파일 변경 명령 preview/diff, 샌드박스 CPU/메모리/PID 제한, Docker socket·주요 시스템 경로 접근 차단. eBPF·fanotify 기반 정교한 감시는 Phase 3.

---

## 9. AI CLI 명령어 설계

### 9.1 기본 구조

```bash
ai [command] [options] [input]
```

### 9.2 주요 서브커맨드

| 명령 | 설명 | 예시 |
|---|---|---|
| `ai ask` | 자연어 질문 답변 | `ai ask "열려 있는 포트 확인 방법"` |
| `ai cmd` | 자연어 → 쉘 명령 변환 | `ai cmd "7일 지난 .log 찾아줘"` |
| `ai run` | 생성 명령 확인 후 실행 | `ai run "node_modules 삭제"` |
| `ai run --preview` | 실행 전 preview 강제 | `ai run --preview "console.log 제거"` |
| `ai preview` | 변경 결과 diff 미리보기 | `ai preview "foo를 bar로 치환"` |
| `ai explain` | 명령/출력 설명 | `ai explain "ps aux \| grep nginx"` |
| `ai fix` | 직전 에러 분석·수정 제안 | `ai fix last-error` |
| `ai summarize` | 파일·로그·출력 요약 | `ai summarize /var/log/nginx/error.log` |
| `ai review-command` | 실행 전 위험성 분석 | `ai review-command "sudo chmod -R 777 /"` |
| `ai script` | 반복 작업 스크립트 생성 | `ai script "매일 새벽 로그 압축"` |
| `ai history` | AI/쉘 기록 조회 | `ai history --ai-only --failed` |
| `ai policy` | 정책 프로파일 조회/전환 | `ai policy set paranoid` |
| `ai doctor` | 진단 | `ai doctor --sandbox` |
| `ai recall` | 이전 세션 작업 검색 | `ai recall "어제 배포 작업"` |
| `ai skill` | 스킬 조회/설치/활성화 (§26) | `ai skill search "도커 취약점"` |
| `ai mcp` | MCP 서버·도구 관리 (§27) | `ai mcp tools github` |
| `ai remote` | 원격 페어링·디바이스 관리 (§28) | `ai remote pair` |

### 9.3 `ai run` 실행 전 표시

```text
Generated command: rm -rf ./node_modules
Risk level: Medium
Reason: Deletes a large dependency directory, but path is scoped.
Execute? [y/N]
```

### 9.4 Dry-run / Diff 미리보기

파일 변경 명령(`sed -i`, `perl -pi`, `awk` 재작성, `python -c` 수정, formatter, codemod, 설정 자동 수정, 대량 rename, AI 생성 스크립트)은 실행 전 diff 표시를 지원한다. 가능한 경우 임시 복사본에서 실행한다.

```text
Proposed command: sed -i 's/foo/bar/g' config.txt
Preview diff:
--- config.txt
+++ config.txt
-foo_enabled=true
+bar_enabled=true
Risk level: Medium   Apply changes? [y/N]
```

명령 자체가 dry-run을 지원하면 우선 사용한다.

| 도구 | Dry-run |
|---|---|
| rsync | `--dry-run` |
| git clean | `-n` |
| terraform | `plan` |
| kubectl | `--dry-run=client` |
| npm | 일부 명령 `--dry-run` |
| rm | 직접 없음 → 대상 목록 미리 출력 |

AI는 destructive command보다 dry-run 대안을 먼저 제안해야 한다.

### 9.5 AI 응답 포맷

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

삭제·변경 명령은 반드시 영향 범위를 설명한다.

---

## 11. 세션 컨텍스트 & Context Stack

수집 가능한 컨텍스트: 현재 cwd, OS 정보, 쉘 종류, 최근 명령/출력/에러, Git 상태, 프로젝트 파일 구조, 패키지 매니저, Docker/Kubernetes 환경, 사용자 정책.

수집 원칙:

- 필요한 최소 정보만 수집한다.
- 민감 파일은 기본 제외한다.
- 사용자가 명시 허용한 경우에만 파일 내용을 읽는다.
- AI 요청 전 컨텍스트 범위를 표시할 수 있다.
- 조직 정책에 따라 외부 AI 전송을 차단할 수 있다.

Context Stack 구조 및 프롬프트 표시는 §7.5 참조.

---

## 17. 권한 모델

| 권한 | 설명 | 매핑 위험 등급(§10.1) |
|---|---|---|
| ReadOnly | 파일 조회, 로그 조회 | Low |
| FileWrite | 파일 생성·수정·삭제 | Medium |
| ProcessControl | 프로세스 종료, 서비스 재시작 | Medium |
| Network | 외부 통신 | High |
| Privileged | sudo, 시스템 설정 변경 | High |
| Destructive | 디스크 포맷, 대량 삭제 | Critical |

AI 생성 명령은 기본적으로 낮은 권한으로 시작하며, 높은 권한이 필요하면 사용자 승인을 받는다. sudo 처리 상세는 §29.12 참조.

---

## 18. 일반 리눅스 터미널 호환성

필수 호환 기능: 표준 입출력/에러 스트림, ANSI escape sequence, TTY/PTY, 인터랙티브 프로그램(Vim·Nano·htop·less·top), `Ctrl+C`·`Ctrl+D`·`Ctrl+Z`, 파이프·리다이렉션·백그라운드 작업, 환경 변수 유지, 쉘 설정 파일 로딩, SSH 세션.

```bash
cat app.log | grep ERROR | less
vim ~/.bashrc
top
ssh user@example.com
docker exec -it app bash
```

호환성 테스트 대상: bash, zsh, fish / Ubuntu, Debian, Fedora, Arch / WSL, Docker container, SSH remote shell.

---

## 19. 엣지 케이스 처리

### 19.1 Shell Built-in 명령

`cd`, `alias`, `export`, `source`는 외부 프로세스가 아니라 built-in이다. 별도 프로세스에서 실행하면 현재 쉘 상태에 반영되지 않는다. → built-in은 현재 PTY의 쉘 컨텍스트에서 실행하고, preview 단계에서 built-in 여부를 표시하며, `cd` 같은 상태 변경 명령 실행 후 cwd를 동기화한다(§7.3).

### 19.2 인터랙티브 명령

`vim`, `ssh`, `top`, `less`, `mysql`, `psql`은 PTY raw mode로 처리한다. AI 출력 요약 대상에서 전체 화면 버퍼를 제외하고, 사용자가 요청한 경우에만 스크린 버퍼 일부를 컨텍스트로 전달한다. **비밀번호 입력 프롬프트 감지 시 AI 컨텍스트 수집을 중단한다.**

### 19.3 명령어 인젝션 방지

사용자 입력값은 기본 shell-escape 처리한다. 위험 메타문자(`;`, `&&`, `||`, `` ` ``, `$()`, `>`) 사용 시 경고한다. 파일명 기반 명령은 가능한 경우 배열 인자 방식으로 실행한다. AI 생성 명령과 사용자 원본 입력을 분리 저장한다.

### 19.4 심볼릭 링크 및 경로 우회

실행 전 대상 경로를 realpath로 정규화하고, 홈/루트/시스템 디렉터리로 확장되는지 검사한다. 심볼릭 링크를 따라가는 명령은 별도 경고한다. 대량 작업 전 대상 파일 목록을 먼저 표시한다.

### 19.5 프롬프트 인젝션 방어

외부 파일 내용은 신뢰할 수 없는 데이터로 표시한다. 시스템 정책·사용자 명령을 외부 컨텍스트보다 우선한다. "이전 지시를 무시하라" 같은 문구는 데이터로만 취급한다. AI 응답 파서에서 실행 가능한 명령과 설명을 분리한다.

### 19.6 원격/로컬 컨텍스트 혼동

프롬프트에 실행 대상 환경(local/ssh/container/k8s)을 표시한다. AI 명령 생성 시 OS·shell·hostname·cwd를 포함하고, 원격 명령은 로컬/원격 파일 경로를 구분한다(§7.5).

### 19.7 멀티라인 명령 및 Here-document

멀티라인 파서를 제공하고 heredoc 종료 토큰을 감지한다. 실행 전 전체 명령 블록을 표시하며, 일부 라인만 실행되는 상태를 방지한다. 붙여넣기 모드를 감지·경고한다.

### 19.8 대량 출력 Backpressure

출력 스트림에 backpressure를 적용하고 UI 렌더링과 로그 저장을 분리한다. 대량 출력은 ring buffer로 최근 N라인만 유지한다. AI 컨텍스트에는 요약/샘플만 전달한다.

---
