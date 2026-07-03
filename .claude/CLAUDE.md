# auction_harness — 경매 투자분석 하네스

이 프로젝트는 한국 법원경매 물건을 투자 관점에서 분석하는 하네스다.

## 오케스트레이터 트리거

"경매 물건 분석", "경매 투자분석", "법원경매 스캔해서 수익 물건 뽑아줘", "경매 리포트 만들어줘" 등의 요청이 오면
**`auction-orchestrator` 스킬**로 5단계 파이프라인(수집→권리분석→시세평가→수익성→랭킹)을 실행한다.

## 에이전트 (`.claude/agents/`)
| 에이전트 | 단계 | 사용 스킬(재사용) |
|----------|------|-------------------|
| auction-collector | 1. 수집 | court-auction-scraper |
| rights-analyzer | 2. 권리분석 | npl-analyzer |
| market-valuator | 3. 시세평가 | realprice-flow, apt-value |
| profitability-calculator | 4. 수익성 | npl-analyzer |
| investment-ranker | 5. 랭킹/리포트 | auction-investment-ranker |

## 데이터 전달
파일 기반. `_workspace/01_listings.csv` → `02_rights_{case}.json` → `03_valuation_{case}.json` → `04_profit_{case}.json` → `05_ranking.json` + `05_report.html`.

## 변경 이력
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-07-03 | 초기 구성 (경매 투자분석 파이프라인, 에이전트 5 + 오케스트레이터 + 랭커 스킬) | 전체 | 하네스 신규 구축 |
