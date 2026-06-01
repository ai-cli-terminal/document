# AI CLI 통합 리눅스 터미널 — 설계 문서 세트

일반 리눅스 터미널 호환성을 최우선으로 유지하면서 AI 보조 기능(명령 생성·설명·디버깅·로그 분석·자동화)을 안전하게 결합하는 통합 터미널의 설계 문서 모음이다. 단일 통합 문서(v3.3)를 카테고리별로 분리했으며, **섹션 번호(§0~§32)는 통합본과 동일하게 유지**하여 문서 간 상호참조가 그대로 유효하다.

| 항목 | 내용 |
|---|---|
| 버전 | v3.3 (통합 + 보강 + 요구사항 3종 + 결정안 + MVP 구현 명세) |
| 상태 | MVP spec finalized — ready to build |
| 최종 정리일 | 2026-06-01 |
| 구성 | README + 7개 카테고리 문서 |

## 문서 구성

| # | 문서 | 내용 | 대상 독자 |
|---|---|---|---|
| 00 | [개요·핵심 원칙·전체 아키텍처](./00-overview-architecture.md) | 검토 요약, 목표, 17개 원칙, 시나리오, 5계층 아키텍처, 결론 | 전원 (진입점) |
| 01 | [핵심 컴포넌트·런타임 설계](./01-core-design.md) | 컴포넌트 상세, Context Consistency, Guardrails, 명령어, 세션 컨텍스트, 권한, 호환성, 엣지케이스 | 코어 구현 |
| 02 | [보안·프라이버시·정책](./02-security-policy.md) | 위험 분류, Zero-Trust, Secret/PII 마스킹, 감사, 정책 프로파일·조직 정책 | 보안 |
| 03 | [서브시스템: 스킬·MCP·리모트](./03-subsystems-skill-mcp-remote.md) | 통합 스킬 관리, 통합 MCP 관리, 앱/웹/모바일 리모트 | 서브시스템 구현 |
| 04 | [설정·저장·운영·테스트](./04-config-ops-testing.md) | 설정 레퍼런스, 플러그인, 저장, 에러/타임아웃, 성능, AI 모드, 테스트, 운영성, 기술 스택 | 플랫폼/운영 |
| 05 | [로드맵·보강점·결정안](./05-roadmap-enhancements-decisions.md) | Phase 로드맵, 추가 보강점 15종, 구현 착수 13개 결정안 | PM/리드 |
| 06 | [MVP 구현 명세](./06-mvp-implementation-spec.md) | 스키마·점수표·룰셋·수용 기준 (구현·검수 계약) | MVP 구현/검수 |

## 권장 읽는 순서

1. **처음 보는 경우**: 00 → 01 → 02 → 03
2. **MVP 구현 착수**: 00(§3 원칙) → 05(§30 결정안) → 06(MVP 명세) → 04(설정·저장)
3. **보안 검토**: 02 → 06(§31.4 위험도, §31.8 마스킹) → 01(§8 Guardrails)
4. **로드맵/범위 협의**: 05 → 03(서브시스템 Phase)

## 섹션 → 파일 매핑 (상호참조 색인)

문서 본문의 `§N` 참조는 아래 표로 해당 파일을 찾는다.

| 섹션 | 제목 | 파일 |
|---|---|---|
| §0 | 설계 검토 요약 | [`00-overview-architecture.md`](./00-overview-architecture.md) |
| §1 | 개요 | [`00-overview-architecture.md`](./00-overview-architecture.md) |
| §2 | 설계 목표 | [`00-overview-architecture.md`](./00-overview-architecture.md) |
| §3 | 핵심 설계 원칙 | [`00-overview-architecture.md`](./00-overview-architecture.md) |
| §4 | 사용자 시나리오 | [`00-overview-architecture.md`](./00-overview-architecture.md) |
| §5 | 전체 아키텍처 | [`00-overview-architecture.md`](./00-overview-architecture.md) |
| §6 | 컴포넌트 상세 설계 | [`01-core-design.md`](./01-core-design.md) |
| §7 | Context Consistency Manager | [`01-core-design.md`](./01-core-design.md) |
| §8 | Execution Guardrails Engine | [`01-core-design.md`](./01-core-design.md) |
| §9 | AI CLI 명령어 설계 | [`01-core-design.md`](./01-core-design.md) |
| §10 | 보안 및 프라이버시 설계 | [`02-security-policy.md`](./02-security-policy.md) |
| §11 | 세션 컨텍스트 & Context Stack | [`01-core-design.md`](./01-core-design.md) |
| §12 | 정책 엔진·프로파일·조직 정책 | [`02-security-policy.md`](./02-security-policy.md) |
| §13 | 통합 설정 파일 레퍼런스 | [`04-config-ops-testing.md`](./04-config-ops-testing.md) |
| §14 | 플러그인 구조 | [`04-config-ops-testing.md`](./04-config-ops-testing.md) |
| §15 | 데이터 저장 구조 | [`04-config-ops-testing.md`](./04-config-ops-testing.md) |
| §16 | 에러 처리·타임아웃·자가 치유 | [`04-config-ops-testing.md`](./04-config-ops-testing.md) |
| §17 | 권한 모델 | [`01-core-design.md`](./01-core-design.md) |
| §18 | 일반 리눅스 터미널 호환성 | [`01-core-design.md`](./01-core-design.md) |
| §19 | 엣지 케이스 처리 | [`01-core-design.md`](./01-core-design.md) |
| §20 | 성능 및 리소스 제어 | [`04-config-ops-testing.md`](./04-config-ops-testing.md) |
| §21 | 로컬/원격/하이브리드 AI | [`04-config-ops-testing.md`](./04-config-ops-testing.md) |
| §22 | 테스트 전략 | [`04-config-ops-testing.md`](./04-config-ops-testing.md) |
| §23 | 운영성 | [`04-config-ops-testing.md`](./04-config-ops-testing.md) |
| §24 | 구현 기술 스택 | [`04-config-ops-testing.md`](./04-config-ops-testing.md) |
| §25 | MVP+ 및 Phase 로드맵 | [`05-roadmap-enhancements-decisions.md`](./05-roadmap-enhancements-decisions.md) |
| §26 | 통합 스킬 관리 | [`03-subsystems-skill-mcp-remote.md`](./03-subsystems-skill-mcp-remote.md) |
| §27 | 통합 MCP 관리 | [`03-subsystems-skill-mcp-remote.md`](./03-subsystems-skill-mcp-remote.md) |
| §28 | 앱/웹/모바일 리모트 | [`03-subsystems-skill-mcp-remote.md`](./03-subsystems-skill-mcp-remote.md) |
| §29 | 추가 보강점 | [`05-roadmap-enhancements-decisions.md`](./05-roadmap-enhancements-decisions.md) |
| §30 | 구현 착수 결정안 | [`05-roadmap-enhancements-decisions.md`](./05-roadmap-enhancements-decisions.md) |
| §31 | MVP 구현 명세 | [`06-mvp-implementation-spec.md`](./06-mvp-implementation-spec.md) |
| §32 | 결론 | [`00-overview-architecture.md`](./00-overview-architecture.md) |

## 핵심 설계 원칙 (요약)

1. 일반 쉘 호환성이 AI 기능보다 우선한다.
2. AI는 보조자이며 최종 실행 권한은 사용자에게 있다.
3. AI 기능 장애는 터미널 장애로 전파되지 않는다.
4. PTY 상태와 AI 컨텍스트는 항상 동기화된다.
5. 모든 외부 컨텍스트는 Zero-Trust Pipeline을 통과하고, Secret/PII는 원격 호출 전 마스킹한다.
6. 파일 변경 명령은 preview·dry-run·diff를 우선 제공한다.
7. AI 생성 명령은 자동 실행하지 않는다.

전체 17개 원칙은 [00-overview-architecture.md §3](./00-overview-architecture.md) 참조.

## MVP 한 줄 요약

Hook 기반 셸 통합(+Wrapper fallback) · 데몬 없음(SQLite WAL+파일락) · 로컬 정책 우선 위험도(0~100) · 파일 수정 preview/diff · best-effort undo · Secret/PII 마스킹 기본 ON · provider capability map. **MCP·원격 릴레이·원격 승인은 MVP 제외.** 상세는 [06-mvp-implementation-spec.md](./06-mvp-implementation-spec.md).
