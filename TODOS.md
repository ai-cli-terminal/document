# TODOS

Deferred work tracked from reviews. Each item has enough context to pick up cold.

## remote-approval (from /plan-ceo-review 2026-06-02)

### T-RA1 — 결과 승인 (outcome approval via diff preview)  [P2]
- **What:** 명령 문자열이 아니라 *결과*를 승인 — §31.5 preview/diff를 승인 payload에 실어 폰에서 "이 변경을 승인".
- **Why:** thesis(D)를 "명령 정확도"에서 "결과 정확도"로 올리는 진짜 10x. 폰에서 실제 변경 내용을 보고 승인.
- **Pros:** 신뢰 체감 최대; 컨텍스트 정확도 thesis의 완성형.
- **Cons / 왜 deferred:** 3-모델 합의(Claude + 2라운드 스펙검수 + Codex)로 보류. (1) §31.5 preview 엔진이 DESIGN 재사용 정본(§31.10/31.4/10.5/28.4/31.8)에 없음 = 미구현, effort = M + 엔진 구축 + previewability 분류기. (2) 분류기가 실질 보안 경계가 됨 → 오분류 위험. (3) 부분 changed-hunks diff가 명령-문자열보다 "거짓 신뢰"를 줄 수 있음(권위 있어 보임). (4) diff 본문이 시크릿/PII 폰 노출 면 확대(§10.4 마스킹 불완전).
- **Context:** 만약 진행하면 — 파이프라인 generate(changed-hunks-only)→§31.8 mask→N줄 truncate→render, §31.5 previewable 서브셋만, 혼합/비결정 명령은 명령-문자열로 강등, 음성 케이스 테스트(추가 라인 시크릿 마스킹 또는 강등). 별도 마일스톤으로, M1 데모가 돈 뒤 재평가.
- **Effort:** human L (M+엔진) / CC ~half-day+
- **Priority:** P2 (M1 데모 이후)
- **Depends on:** M1 데모 green; §31.5 preview 엔진 빌드 가능 확인 먼저.

### T-RA2 — 거부 사유 메모 회신  [P3]
- **What:** Reject 시 폰이 한 줄 사유를 회신 → 터미널에 표시("rejected: wrong branch").
- **Why:** 거부 루프 인간화, 작은 delight.
- **Effort:** human S / CC ~30min · **Priority:** P3

### T-RA3 — 승인 히스토리 타임라인 (PWA)  [P3]
- **What:** PWA가 최근 승인 요청 + 결과를 로컬 보관. "외출 중 뭘 승인했나" 회고.
- **Why:** 감사성/안심. **Cons:** PWA 로컬 상태/스토리지 스코프 증가.
- **Effort:** human S-M / CC ~1h · **Priority:** P3

### T-RA4 — 승인 채널 범용 이벤트 버스로 일반화  [P3]
- **What:** 승인 payload/릴레이를 "터미널→폰 서명 이벤트" 범용 채널로 일반화 → 예산 알림·가드레일 발동·작업 완료 등이 재사용.
- **Why:** §29가 이미 budget/guardrail 푸시를 언급 — 플랫폼 잠재력.
- **Cons / 주의:** 학습 일회성에 대한 **과설계(premature abstraction) 위험.** M2 릴레이가 돈 뒤, 두 번째 이벤트 타입이 실제로 필요해질 때 일반화하라(YAGNI).
- **Effort:** human L / CC ~half-day · **Priority:** P3 · **Depends on:** M2 relay
