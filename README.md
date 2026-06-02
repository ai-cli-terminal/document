# AI CLI 통합 리눅스 터미널 — 설계 문서 세트

일반 리눅스 터미널 호환성을 최우선으로 유지하면서 AI 보조 기능(명령 생성·설명·디버깅·로그 분석·자동화)을 안전하게 결합하는 통합 터미널의 설계 문서 모음이다. 단일 통합 문서(v3.3)를 카테고리별로 분리했으며, **섹션 번호(§0~§32)는 통합본과 동일하게 유지**하여 문서 간 상호참조가 그대로 유효하다.

| 항목 | 내용 |
|---|---|
| 버전 | v3.3 (통합 + 보강 + 요구사항 3종 + 결정안 + MVP 구현 명세) |
| 상태 | MVP spec finalized — ready to build |
| 최종 정리일 | 2026-06-01 |
| 구성 | README + `docs/` 설계 문서 7종 + `planning/` 산출물 17종 + `planning/builds/` 빌드 문서 |

## 저장소 구조

```text
document/
├─ README.md                  ← 이 파일 (진입점)
├─ docs/                       ← 설계 정본 7종 (§0~§32, 상호참조 기준)
└─ planning/                  ← 프로젝트 산출물 17종 (계획·요구사항·운영·테스트)
   └─ builds/<feature>/        ← 기능별 빌드 문서 (DESIGN, TEST-PLAN)
```

## 설계 문서 (`docs/`)

| # | 문서 | 내용 | 대상 독자 |
|---|---|---|---|
| 00 | [개요·핵심 원칙·전체 아키텍처](./docs/00-overview-architecture.md) | 검토 요약, 목표, 20개 원칙, 시나리오, 5계층 아키텍처, 결론 | 전원 (진입점) |
| 01 | [핵심 컴포넌트·런타임 설계](./docs/01-core-design.md) | 컴포넌트 상세, Context Consistency, Guardrails, 명령어, 세션 컨텍스트, 권한, 호환성, 엣지케이스 | 코어 구현 |
| 02 | [보안·프라이버시·정책](./docs/02-security-policy.md) | 위험 분류, Zero-Trust, Secret/PII 마스킹, 감사, 정책 프로파일·조직 정책 | 보안 |
| 03 | [서브시스템: 스킬·MCP·리모트](./docs/03-subsystems-skill-mcp-remote.md) | 통합 스킬 관리, 통합 MCP 관리, 앱/웹/모바일 리모트 | 서브시스템 구현 |
| 04 | [설정·저장·운영·테스트](./docs/04-config-ops-testing.md) | 설정 레퍼런스, 플러그인, 저장, 에러/타임아웃, 성능, AI 모드, 테스트, 운영성, 기술 스택 | 플랫폼/운영 |
| 05 | [로드맵·보강점·결정안](./docs/05-roadmap-enhancements-decisions.md) | Phase 로드맵, 추가 보강점 15종, 구현 착수 13개 결정안 | PM/리드 |
| 06 | [MVP 구현 명세](./docs/06-mvp-implementation-spec.md) | 스키마·점수표·룰셋·수용 기준 (구현·검수 계약) | MVP 구현/검수 |

## 프로젝트 산출물 (`planning/`)

설계 정본(`docs/`)을 바탕으로 한 실행 산출물이다. 위험도·정책·DDL 등의 권위값은 `docs/`(특히 §31 MVP 명세)를 정본으로 참조한다.

| # | 문서 | 내용 |
|---|---|---|
| 01 | [프로젝트 계획서](./planning/01_프로젝트_계획서.md) | 배경·목표·범위·KPI·위험도 기준 |
| 02 | [ERD 문서](./planning/02_ERD_문서.md) | SQLite `ai-terminal.db` 7테이블 스키마·DDL |
| 03 | [아키텍처 정의서](./planning/03_프로젝트_아키텍처_정의서.md) | 5계층 + Remote Gateway, ADR 5건 |
| 04 | [API 명세서](./planning/04_API_명세서.md) | `ai` CLI 명령·내부 인터페이스·종료 코드 |
| 05 | [화면 흐름 시퀀스 다이어그램](./planning/05_화면_흐름_시퀀스_다이어그램.md) | 핵심 사용자 플로우 시퀀스 |
| 06 | [화면 기능 정의서](./planning/06_화면_기능_정의서.md) | TUI 화면 14종 (SCR-T-* ID) |
| 07 | [요구사항 정의서](./planning/07_요구사항_정의서.md) | FR/NFR 93개, 우선순위 |
| 08 | [스토리 보드](./planning/08_스토리_보드.md) | 페르소나 4종 Epic→Story→AC |
| 09 | [Git 규칙 정의서](./planning/09_Git_규칙_정의서.md) | GitHub Flow, Conventional Commits, 릴리즈 |
| 10 | [환경 설정 템플릿](./planning/10_환경_설정_템플릿.md) | `config.toml` 정본, 빌드 환경, 디렉터리 |
| 11 | [테스트 전략서](./planning/11_테스트_전략서.md) | 테스트 피라미드, 커버리지 목표, CI |
| 12 | [코드 리뷰 규칙](./planning/12_코드_리뷰_규칙.md) | 리뷰 역할·프로세스·체크리스트 |
| 13 | [테스트 보고서](./planning/13_테스트_보고서.md) | 마일스톤별 결과 템플릿(측정 후 기입) |
| 14 | [배포 가이드](./planning/14_배포_가이드.md) | 빌드→서명→릴리즈→설치, 롤백 |
| 15 | [사용자 메뉴얼](./planning/15_사용자_메뉴얼.md) | 설치·셸 통합·정책·FAQ |
| 16 | [운영 메뉴얼](./planning/16_운영_메뉴얼.md) | 모니터링·장애 플레이북·백업·거버넌스 |
| 17 | [스케줄](./planning/17_스케줄.md) | Phase 1(16주) 마일스톤, Gantt |

## 빌드 문서 (`planning/builds/`)

기능 단위 설계·테스트 문서. `/office-hours`·`/plan-eng-review` 산출물이며 `docs/` 정본을 입력으로 쓴다.

| 기능 | 문서 |
|---|---|
| remote-approval | [DESIGN](./planning/builds/remote-approval/DESIGN.md) · [TEST-PLAN](./planning/builds/remote-approval/TEST-PLAN.md) |

## 권장 읽는 순서

1. **처음 보는 경우**: 00 → 01 → 02 → 03
2. **MVP 구현 착수**: 00(§3 원칙) → 05(§30 결정안) → 06(MVP 명세) → 04(설정·저장)
3. **보안 검토**: 02 → 06(§31.4 위험도, §31.8 마스킹) → 01(§8 Guardrails)
4. **로드맵/범위 협의**: 05 → 03(서브시스템 Phase)

## 섹션 → 파일 매핑 (상호참조 색인)

문서 본문의 `§N` 참조는 아래 표로 해당 파일을 찾는다.

| 섹션 | 제목 | 파일 |
|---|---|---|
| §0 | 설계 검토 요약 | [`00-overview-architecture.md`](./docs/00-overview-architecture.md) |
| §1 | 개요 | [`00-overview-architecture.md`](./docs/00-overview-architecture.md) |
| §2 | 설계 목표 | [`00-overview-architecture.md`](./docs/00-overview-architecture.md) |
| §3 | 핵심 설계 원칙 | [`00-overview-architecture.md`](./docs/00-overview-architecture.md) |
| §4 | 사용자 시나리오 | [`00-overview-architecture.md`](./docs/00-overview-architecture.md) |
| §5 | 전체 아키텍처 | [`00-overview-architecture.md`](./docs/00-overview-architecture.md) |
| §6 | 컴포넌트 상세 설계 | [`01-core-design.md`](./docs/01-core-design.md) |
| §7 | Context Consistency Manager | [`01-core-design.md`](./docs/01-core-design.md) |
| §8 | Execution Guardrails Engine | [`01-core-design.md`](./docs/01-core-design.md) |
| §9 | AI CLI 명령어 설계 | [`01-core-design.md`](./docs/01-core-design.md) |
| §10 | 보안 및 프라이버시 설계 | [`02-security-policy.md`](./docs/02-security-policy.md) |
| §11 | 세션 컨텍스트 & Context Stack | [`01-core-design.md`](./docs/01-core-design.md) |
| §12 | 정책 엔진·프로파일·조직 정책 | [`02-security-policy.md`](./docs/02-security-policy.md) |
| §13 | 통합 설정 파일 레퍼런스 | [`04-config-ops-testing.md`](./docs/04-config-ops-testing.md) |
| §14 | 플러그인 구조 | [`04-config-ops-testing.md`](./docs/04-config-ops-testing.md) |
| §15 | 데이터 저장 구조 | [`04-config-ops-testing.md`](./docs/04-config-ops-testing.md) |
| §16 | 에러 처리·타임아웃·자가 치유 | [`04-config-ops-testing.md`](./docs/04-config-ops-testing.md) |
| §17 | 권한 모델 | [`01-core-design.md`](./docs/01-core-design.md) |
| §18 | 일반 리눅스 터미널 호환성 | [`01-core-design.md`](./docs/01-core-design.md) |
| §19 | 엣지 케이스 처리 | [`01-core-design.md`](./docs/01-core-design.md) |
| §20 | 성능 및 리소스 제어 | [`04-config-ops-testing.md`](./docs/04-config-ops-testing.md) |
| §21 | 로컬/원격/하이브리드 AI | [`04-config-ops-testing.md`](./docs/04-config-ops-testing.md) |
| §22 | 테스트 전략 | [`04-config-ops-testing.md`](./docs/04-config-ops-testing.md) |
| §23 | 운영성 | [`04-config-ops-testing.md`](./docs/04-config-ops-testing.md) |
| §24 | 구현 기술 스택 | [`04-config-ops-testing.md`](./docs/04-config-ops-testing.md) |
| §25 | MVP+ 및 Phase 로드맵 | [`05-roadmap-enhancements-decisions.md`](./docs/05-roadmap-enhancements-decisions.md) |
| §26 | 통합 스킬 관리 | [`03-subsystems-skill-mcp-remote.md`](./docs/03-subsystems-skill-mcp-remote.md) |
| §27 | 통합 MCP 관리 | [`03-subsystems-skill-mcp-remote.md`](./docs/03-subsystems-skill-mcp-remote.md) |
| §28 | 앱/웹/모바일 리모트 | [`03-subsystems-skill-mcp-remote.md`](./docs/03-subsystems-skill-mcp-remote.md) |
| §29 | 추가 보강점 | [`05-roadmap-enhancements-decisions.md`](./docs/05-roadmap-enhancements-decisions.md) |
| §30 | 구현 착수 결정안 | [`05-roadmap-enhancements-decisions.md`](./docs/05-roadmap-enhancements-decisions.md) |
| §31 | MVP 구현 명세 | [`06-mvp-implementation-spec.md`](./docs/06-mvp-implementation-spec.md) |
| §32 | 결론 | [`00-overview-architecture.md`](./docs/00-overview-architecture.md) |

## 핵심 설계 원칙 (요약)

1. 일반 쉘 호환성이 AI 기능보다 우선한다.
2. AI는 보조자이며 최종 실행 권한은 사용자에게 있다.
3. AI 기능 장애는 터미널 장애로 전파되지 않는다.
4. PTY 상태와 AI 컨텍스트는 항상 동기화된다.
5. 모든 외부 컨텍스트는 Zero-Trust Pipeline을 통과하고, Secret/PII는 원격 호출 전 마스킹한다.
6. 파일 변경 명령은 preview·dry-run·diff를 우선 제공한다.
7. AI 생성 명령은 자동 실행하지 않는다.

전체 20개 원칙은 [00-overview-architecture.md §3](./docs/00-overview-architecture.md) 참조.

## MVP 한 줄 요약

Hook 기반 셸 통합(+Wrapper fallback) · 데몬 없음(SQLite WAL+파일락) · 로컬 정책 우선 위험도(0~100) · 파일 수정 preview/diff · best-effort undo · Secret/PII 마스킹 기본 ON · provider capability map. **MCP·원격 릴레이·원격 승인은 MVP 제외.** 상세는 [06-mvp-implementation-spec.md](./docs/06-mvp-implementation-spec.md).
