# auction_harness — Loop-Engineered Team-Architecture Factory

Claude Code용 **팀 아키텍처 팩토리** 메타 스킬. 도메인 한 문장을 에이전트 팀과 스킬 세트로 변환한다.
[revfactory/harness](https://github.com/revfactory/harness) (Apache-2.0)의 포크로, **루프 엔지니어링**을 접목해 검증(Phase 6)과 진화(Phase 7)를 강화했다.

이 저장소는 두 가지를 담는다:
1. **`skills/harness/`** — 루프 엔지니어링을 접목한 harness 메타 스킬(팩토리).
2. **`.claude/`** — 그 메타 스킬로 실제 생성한 예제 하네스: **법원경매 투자분석 파이프라인**([아래](#예제-하네스--법원경매-투자분석-claude)).

## 이 포크가 추가한 것

### Phase 6 — 검증 수렴 루프 (Convergence Loop)
"한 번 훑고 끝"이던 검증을 **게이트를 통과할 때까지 반복하되 반드시 유한하게 종료**하는 루프로 승격.

- **게이트 지표** (hard/soft 분리): 구조·배선·트리거정확도(hard) / 실행품질·커버리지(soft)
- **정지 조건 4종**: 성공 / 소진(R_max) / 정체(Δ<ε가 S회) / 예산초과 — 무한 루프·비용 폭주 차단
- **예산 배분 규칙**: 고비용 실행 테스트(6-3)는 hard 게이트 통과 후에만, 커버리지 크리틱(6-7)은 별도 예산(C_budget)으로 실행해 기아 방지
- **커버리지 크리틱** (loop-until-dry): "무엇이 빠졌나"를 연속 K회 "없음"까지
- **적대적 트리거 검증** (옵션): 애매한 트리거를 회의론 에이전트 3개 다수결로 판정
- **상태 영속화**: `_workspace/validation_state.json`으로 재개·퇴행 감지

### Phase 7 — 자기개선 루프 (Self-Improvement Loop)
선언적 진화 원칙을 **신호 수집 → 진단 → 승인 게이트 → 적용 → Phase 6 재검증 → 이력** 루프로 구현.

- **텔레메트리 신호 수집**: 명시적 피드백 + 자동 마찰 지표(반복 실패, 게이트 미달, 오케스트레이터 우회)
- **안전장치**: 사람 승인 게이트(자동 변경 금지) · Phase 6 재검증(퇴행 시 롤백) · 원자적 변경 · 무한 자기수정 방지

상세 루프 제어·파라미터·프롬프트 템플릿은 [`skills/harness/references/validation-loop.md`](skills/harness/references/validation-loop.md) 참조.

## 6가지 팀 아키텍처 패턴 (원본 계승)

| 패턴 | 용도 |
|------|------|
| Pipeline | 순차 의존 작업 |
| Fan-out/Fan-in | 병렬 독립 작업 + 결과 집계 |
| Expert Pool | 맥락 기반 선택적 호출 |
| Producer-Reviewer | 생성 후 품질 검토 |
| Supervisor | 중앙 조정자의 동적 분배 |
| Hierarchical Delegation | 상향식 재귀 위임 |

## 설치

글로벌 스킬로 복사:
```bash
cp -R skills/harness ~/.claude/skills/harness
```

또는 플러그인으로:
```
/plugin marketplace add earthskyisbig/auction_harness
/plugin install auction-harness@auction_harness
```

## 사용

```
"이 프로젝트에 맞는 하네스 구성해줘"
"하네스 설계해줘"
"하네스 점검해줘"   # 운영/유지보수
```

## 예제 하네스 — 법원경매 투자분석 (`.claude/`)

메타 스킬로 생성한 실동작 하네스. 한국 법원경매 물건을 **수집 → 권리분석 → 시세평가 → 수익성계산 → 투자등급 랭킹**의 5단계 파이프라인으로 분석한다. (Pipeline + 물건별 Fan-out)

### 사용
`.claude/`가 있는 프로젝트에서:
```
"서울 강남구 아파트 경매 물건 분석해줘"
"법원경매 스캔해서 수익 나는 물건 뽑아줘"
"경매 투자 리포트 만들어줘"
```
→ `auction-orchestrator`가 파이프라인을 돌려 `_workspace/05_report.html`(S~D 등급 리포트)을 생성한다.

### 에이전트 (`.claude/agents/`, 모두 opus)
| 에이전트 | 단계 | 재사용 스킬 |
|----------|------|-------------|
| `auction-collector` | 1. 수집 | court-auction-scraper |
| `rights-analyzer` | 2. 권리분석 | npl-analyzer |
| `market-valuator` | 3. 시세평가 | realprice-flow, apt-value |
| `profitability-calculator` | 4. 수익성 | npl-analyzer |
| `investment-ranker` | 5. 랭킹/리포트 | auction-investment-ranker |

### 스킬 (`.claude/skills/`)
- **`auction-orchestrator`** — 5단계 파이프라인 오케스트레이터. 2~4단계는 물건별 fan-out(병렬), 5단계에서 집계 barrier.
- **`auction-investment-ranker`** — S~D 채점 루브릭(`ROI×0.6 − 리스크×0.4`) + HTML 리포트 템플릿.

### 데이터 흐름 (파일 기반, `_workspace/`)
```
01_listings.csv → 02_rights_{건}.json → 03_valuation_{건}.json
              → 04_profit_{건}.json → 05_ranking.json + 05_report.html
```

### 설계 원칙
- **기존 스킬 재사용** — court-auction-scraper·npl-analyzer·realprice-flow·apt-value를 재사용하고 랭커 스킬만 신규 생성(중복 방지).
- **보수적 판단** — 권리 애매 시 인수위험 상향, 리스크플래그 2개↑ 물건은 자동 강등.
- **투명성** — data_gap·미분석 물건을 리포트에 반드시 노출.

> ⚠️ 리포트는 실거래·감정가·권리 추정 기반 참고자료이며, 투자 판단과 손실 책임은 투자자에게 있다.

## 구조

```
auction_harness/
├── .claude-plugin/plugin.json
├── LICENSE                       # Apache-2.0 (원본 계승)
├── NOTICE                        # 귀속 표기
├── skills/harness/               # ① harness 메타 스킬 (팩토리)
│   ├── SKILL.md                  #    7-phase 워크플로우 (Phase 6/7 루프 강화)
│   └── references/
│       ├── validation-loop.md    #    ★ 신규: 검증 루프 상세
│       ├── agent-design-patterns.md
│       ├── orchestrator-template.md
│       ├── skill-writing-guide.md
│       ├── skill-testing-guide.md
│       ├── qa-agent-guide.md
│       └── team-examples.md
└── .claude/                      # ② 예제: 법원경매 투자분석 하네스
    ├── CLAUDE.md                 #    오케스트레이터 트리거 + 변경이력
    ├── agents/                   #    수집·권리·시세·수익·랭킹 (5개)
    └── skills/
        ├── auction-orchestrator/
        └── auction-investment-ranker/
```

## 라이선스 / 귀속

Apache-2.0. 원저작물: [revfactory/harness](https://github.com/revfactory/harness) by robin. 변경 사항은 [`NOTICE`](NOTICE) 참조.
