# 검증 루프 상세 가이드 (Validation Loop)

Phase 6(검증 및 테스트)의 수렴 루프를 실행하기 위한 상세 레퍼런스.
SKILL.md 본문은 요약만 담고, 파라미터·스키마·프롬프트 템플릿은 이 문서에서 관리한다.

핵심 명제: **검증은 게이트를 통과할 때까지 반복하되, 정지 조건과 예산으로 반드시 유한하게 멈춘다.**
정지 조건이 없는 루프, 예산이 없는 루프는 금지한다.

---

## 1. 게이트 지표 정의

| 게이트 키 | 측정 출처 | 산출 방법 | 통과 판정 |
|-----------|-----------|-----------|-----------|
| `structure` | 6-1 | 구조 오류 개수 | 오류 == 0 (hard) |
| `wiring` | 6-2 | dead-link 개수 | dead-link == 0 (hard) |
| `trigger` | 6-4 | (should-trigger 통과율, should-NOT 통과율) | 둘 다 ≥ 임계 (hard) |
| `exec` | 6-3 | with-skill 품질 0~10, baseline 대비 우위 | 점수 ≥ 임계 AND 우위 (soft) |
| `coverage` | 6-7 | 크리틱 "누락 없음" 연속 횟수 | 연속 ≥ K (soft) |

- **hard 게이트**: 전부 통과해야 "성공 종료" 가능.
- **soft 게이트**: 예산 소진 시 미달이어도 종료 가능하되, 종료 리포트(6-8)에 미달 명시.
- **실행 순서 규칙(예산 배분):**
  - `exec`(6-3)는 고비용이므로 **hard 게이트가 모두 통과한 반복에서만** 측정한다. hard 미통과 반복에서는 건너뛰어 hard 수정에 예산을 집중.
  - `coverage`(6-7)는 R_max와 분리된 `C_budget`으로 실행하며, hard가 처음 전부 통과한 반복부터 streak을 센다.

---

## 2. 도메인 난이도별 프리셋

게이트 임계값·루프 파라미터는 도메인 복잡도에 맞춰 조정한다. 아래는 시작점이며 사용자와 합의해 조정.

| 파라미터 | 단순(simple) | 표준(standard) | 복잡(complex) |
|----------|-------------|----------------|---------------|
| trigger 임계 (should / should-NOT) | 85% / 85% | 90% / 90% | 90% / 95% |
| exec 품질 임계 (0~10) | 6 | 7 | 8 |
| R_max (최대 반복) | 2 | 3 | 5 |
| S (정체 판정 연속 횟수) | 1 | 2 | 2 |
| ε (개선 델타 하한, 점) | 1.5 | 2.0 | 2.0 |
| N_iter (반복당 검증 에이전트 상한) | 4 | 6 | 10 |
| K (커버리지 "없음" 연속 목표) | 1 | 2 | 2 |
| C_budget (커버리지 별도 재진입 예산) | 1 | 1 | 2 |

**난이도 판정 힌트**
- simple: 스킬 1~2개, 단일 실행 모드, 외부 의존 없음
- standard: 스킬 3~5개, 파이프라인/팬아웃, 데이터 전달 1~2회
- complex: 하이브리드 모드, 에이전트 6+개, Phase 경계 다수, 외부 API/파일 의존

---

## 3. 정지 조건 (하나라도 참이면 종료)

```
STOP if any:
  (성공)   모든 hard 게이트 통과  [+ 가능하면 soft 게이트도]
  (소진)   iteration >= R_max
  (정체)   전체 점수 개선 Δ < ε 가 S회 연속
  (예산)   spent_agents >= R_max * N_iter  또는 토큰 예산 초과
```

- **전체 점수(scalar)** 계산 예: `score = 100*structure_ok + 100*wiring_ok + 100*trigger_avg + 10*exec` 같은 가중합.
  절대값보다 **반복 간 델타(Δ)** 추적이 목적이므로 일관된 공식이면 됨.
- 성공이 아닌 종료(소진/정체/예산)는 **"미수렴 종료"** 로 분류하고 미달 게이트를 반드시 보고.

---

## 4. 루프 제어 상태 스키마 (`_workspace/validation_state.json`)

```json
{
  "domain_tier": "standard",
  "params": { "R_max": 3, "S": 2, "epsilon": 2.0, "N_iter": 6, "K": 2,
              "trigger_threshold": 0.90, "exec_threshold": 7 },
  "iteration": 2,
  "history": [
    { "iter": 1, "gates": {"structure": true, "wiring": true,
        "trigger": 0.80, "exec": 6.0, "coverage": 0},
      "score": 386.0, "delta": null, "no_improve_streak": 0,
      "spent_agents": 5,
      "open_issues": [
        {"id":"T1","target":"skills/foo","type":"trigger","desc":"'~로 뽑아줘' 표현 미트리거","severity":"hard"},
        {"id":"E1","target":"skills/bar","type":"exec","desc":"분석 깊이 부족","severity":"soft"}
      ] },
    { "iter": 2, "gates": {"structure": true, "wiring": true,
        "trigger": 0.93, "exec": 7.5, "coverage": 1},
      "score": 468.0, "delta": 82.0, "no_improve_streak": 0,
      "spent_agents": 9,
      "open_issues": [
        {"id":"E1","target":"skills/bar","type":"exec","desc":"분석 깊이 부족","severity":"soft"}
      ] }
  ],
  "termination": null
}
```

`termination` 값: `"success" | "exhausted" | "stalled" | "budget"` (종료 시 채움).

### Resume(재개)
- 루프가 중단되면 이 파일을 읽어 `iteration`, `history[-1]`, `open_issues`부터 이어서 진행한다.
- 재개 시 이전 게이트 점수를 재측정하여 퇴행(regression)이 없는지 먼저 확인한다.

---

## 5. 이슈 우선순위 (6-L 수정 단계)

open_issues 처리 순서:
1. `severity: hard` 이슈 우선 (구조/배선/트리거).
2. 같은 대상에 여러 이슈면 묶어서 한 번에 수정(중복 재테스트 방지).
3. **일반화 원칙**: 특정 테스트 케이스만 통과시키는 좁은 수정 금지. 규칙 수준으로 일반화.
4. 수정 대상 매핑은 Phase 7-2 "피드백 반영 경로" 표를 그대로 재사용:
   - 결과물 품질 → 해당 스킬 / 에이전트 역할 → 에이전트 `.md` / 워크플로우 → 오케스트레이터 / 트리거 → description.

---

## 6. 적대적 트리거 검증 (6-4 옵션)

통과가 애매한 트리거(경계 near-miss에서 흔들리는 케이스)는 회의론 에이전트로 교차 판정한다.

**절차**
1. 애매 케이스(should-trigger인데 불확실 / should-NOT인데 트리거될 뻔한 것)를 선별.
2. 회의론 에이전트 3개를 병렬 스폰. 각 에이전트에 아래 프롬프트를 준다.
3. 다수결(2/3 이상 "오작동")이면 실패로 확정 → open_issues 등록.

**회의론 에이전트 프롬프트 템플릿**
```
너는 스킬 트리거 검증관이다. 아래 스킬 description과 사용자 쿼리가 주어진다.
이 쿼리에서 스킬이 "잘못 트리거되거나 / 잘못 누락되는" 근거를 최대한 찾아 반박하라.
불확실하면 기본값을 "오작동(문제 있음)"으로 둔다. 확신할 때만 "정상"이라고 답하라.

[스킬 description]: {{description}}
[쿼리]: {{query}}
[기대]: should-trigger | should-NOT-trigger

출력(JSON): { "verdict": "정상" | "오작동", "reason": "...", "confidence": 0.0~1.0 }
```

**주의:** 적대적 검증은 비용이 크므로 애매 케이스에만 적용. 명백한 통과/실패엔 쓰지 않는다.

---

## 7. 커버리지 크리틱 (6-7, loop-until-dry)

**크리틱 에이전트 프롬프트 템플릿**
```
너는 하네스 검증 완결성 크리틱이다. 아래 산출물과 지금까지의 검증 결과가 주어진다.
"아직 검증되지 않았거나 누락된 것"만 찾아라. 없으면 정확히 "누락 없음"이라고 답하라.

점검 축:
- 검증 안 된 트리거 표현(사용자가 쓸 법한데 테스트에 없는 것)
- 다루지 않은 실행 시나리오(에러 흐름 포함)
- 배선 안 된 데이터 경로(Phase 경계 dead link)
- 누락된 에러/폴백 케이스

[산출물 요약]: {{artifacts}}
[검증 이력]: {{history}}

출력(JSON): { "gaps": [ {"type":"...","desc":"...","severity":"hard|soft"} ] }  // 없으면 gaps: []
```

**루프 규칙**
- **카운트 시작:** hard 게이트(structure·wiring·trigger)가 처음으로 전부 통과한 반복부터 `coverage` streak을 센다. 그 전에는 크리틱을 실행하지 않는다 — hard 수정 과정에서 나온 gap은 어차피 재변동되어 낭비다.
- `gaps`가 비어있음 → `coverage` 연속 카운트 +1. 연속 K회면 커버리지 게이트 통과.
- `gaps`가 있음 → open_issues에 추가하고 재진입한다. **단 R_max와 분리된 `C_budget`(기본 1회)만 사용**한다. 이 별도 예산이 hard 게이트 수정에 쓰이지 않으므로 커버리지 기아(starvation)를 방지한다.
- `C_budget` 소진 시 streak이 K 미만이어도 soft 게이트로서 미달을 6-8 리포트에 명시하고 종료한다(hard가 green이면 실사용 가능).
- 크리틱이 반복적으로 사소한 트집만 낸다면 severity가 soft인 gap은 예산 소진 시 무시 가능.

---

## 8. 안티패턴 (하지 말 것)

- 정지 조건 없이 "만족할 때까지" 반복 → 무한/비용 폭주.
- 예산 카운터 없이 에이전트 무제한 스폰.
- 특정 테스트 케이스만 통과시키는 과적합 수정.
- soft 게이트에 집착해 종료를 무한 지연 (hard만 필수임을 기억).
- 매 반복에서 게이트 재측정을 생략 → 수정이 다른 게이트를 퇴행시켜도 못 잡음.
- validation_state.json 미기록 → resume·퇴행 추적 불가.
