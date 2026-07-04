# auction_harness — 경매 투자분석 하네스

이 프로젝트는 한국 법원경매 물건을 투자 관점에서 분석하는 하네스다.

## 오케스트레이터 트리거

"경매 물건 분석", "경매 투자분석", "법원경매 스캔해서 수익 물건 뽑아줘", "경매 리포트 만들어줘" 등의 요청이 오면
**`auction-orchestrator` 스킬**로 5단계 파이프라인(수집→권리분석→시세평가→수익성→랭킹)을 실행한다.

## 에이전트 (`.claude/agents/`)
| 에이전트 | 단계 | 사용 스킬(재사용) |
|----------|------|-------------------|
| auction-collector | 1. 수집 | 번들 `scrape_auction_filtered.py` (court-auction-scraper 패턴 기반, 지역·용도 필터) |
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
| 2026-07-04 | 수집 단계에 템플릿 기반 필터 크롤러 번들(`scrape_auction_filtered.py` + templates/) | agents/auction-collector, skills/auction-orchestrator/scripts | 기본 스크래퍼가 지역·용도 서버필터 불가(기본 100건만) → 검증된 일반화 크롤러로 교체. 단지명·동·호·저감율·매각기일까지 수집 |
| 2026-07-04 | 크롤러 다중 페이지그룹 수집(100건 한도 극복) + 시/군/구 완전성 한계 문서화 | skills/auction-orchestrator/scripts/scrape_auction_filtered.py | 100건 초과 수집 필요 사례(서울남부 다세대) 발견. nextPage_btn으로 그룹 순회. 단, 시/군/구 서버필터 미적용은 근본 한계(사후 필터만) — 소규모 정밀검색은 사이트가 정답 |
