# 세션 기록: harness 루프엔지니어링 → 경매 투자분석 파이프라인 구축  (2026-07-03 ~ 2026-07-05)

## 1. 개요
- **주제**: revfactory/harness 분석 → 루프 엔지니어링 접목 → 내 GitHub에 경매 투자분석 하네스 구축 → 실제 물건 발굴·시세·권리분석
- **목표**: (1) harness 메타 스킬에 검증/진화 루프 반영, (2) 법원경매 물건을 지역·용도 필터로 수집→시세→권리→투자판단 자동화, (3) 강의화, (4) 세션 기록 스킬(myprompt) 구축
- **작업 디렉토리**: `/Users/leomyung/harness_def` (셸 cwd), 실제 저장소 `/Users/leomyung/auction_harness`
- **저장소**: https://github.com/earthskyisbig/auction_harness (빈 레포 → 본 세션에서 채움)
- **사용 스킬**: harness, court-auction-scraper, court-auction-detail, apt-value, realprice-flow, algo-design

## 2. 프롬프트 → 액션 → 결과 (타임라인)
| # | 사용자 요청(요약) | 한 일 | 산출물 |
|--:|------------------|-------|--------|
| 1 | revfactory/harness 강약점 + 루프/멀티에이전트 검토 | 레포 분석, 강약점·개선안 제시 | 분석 리포트 |
| 2 | Phase 6 루프 패치안 작성 | 수렴 루프 설계 | 패치 문서 |
| 3~5 | 1,2,3 순서로 적용 | `~/.claude/skills/harness` Phase 6/7 루프 재작성 + `references/validation-loop.md` 신규 | 글로벌 harness 스킬 개선 |
| 6 | 드라이런 테스트 | 시나리오 추적 → **결함 2개 발견·보완** | 커버리지 예산분리·실행테스트 조건실행 |
| 7 | 내 GitHub(earthskyisbig/auction_harness)에 반영 | 개선 메타 스킬 발행(Apache-2.0 귀속) | 커밋 `f514272` |
| 8 | 경매 도메인 하네스 생성 | agents 5 + orchestrator + ranker 스킬 | 커밋 `b28c332` |
| 9 | README 경매 섹션 / .env 경로 / 키 | README·.gitignore·.env.example, 국토부 키 세팅 안내 | 커밋 `7b2cfe7`,`db76efe` |
| 10 | 강남/안산/금천 경매 분석 | 수집→단지필터→시세→권리 파이프라인 실행 | 물건 리스트·분석 |
| 11 | scrape_uijeongbu_apt.py를 다른 검색에도 | 기존 자산 발견 → 템플릿 필터 크롤러 통합 | 커밋 `17f0ff0` |
| 12 | 강의화 + 소요시간 | 8모듈+캡스톤 커리큘럼(~14h) | `docs/course-curriculum.md` 커밋 `2762555` |
| 13 | 금천 다세대 24건 vs 내 7건 불일치 | 근본원인 규명 → 다중 페이지그룹 수집 구현 | 커밋 `e900953` |
| 14 | 통과 물건 권리·시세 검토 | 유찰1회 4건 권리스캔 → 통과 2건 심층 | 투자 판정 |
| 15 | myprompt 스킬 + 세션 기록 | 본 스킬·기록 생성 | `~/.claude/skills/myprompt`, 이 파일 |

## 3. ⭐ 에러 & 원인 & 수정 (재발방지 핵심)

| 문제/에러 | 원인 | 수정/해결 | 교훈 |
|-----------|------|-----------|------|
| `command not found: timeout` | macOS엔 GNU `timeout` 없음 | 스크립트 직접 실행(내부 대기 상한 있음) 또는 `gtimeout` | macOS에서 `timeout` 쓰지 말 것 |
| `ModuleNotFoundError: PublicDataReader` | 미설치 | `pip3 install "PublicDataReader>=1.1.1" python-dotenv --break-system-packages` | **1.0.x는 폐기 엔드포인트(127.0.0.1) 호출 → 반드시 ≥1.1.1** |
| 실거래/공동주택 API 403 또는 키 없음 | `PUBLIC_DATA_SERVICE_KEY` 미설정, 공동주택 API는 **별도 활용신청** | `.env`에 Decoding 키, data.go.kr에서 기본정보+목록 서비스 각각 신청 | 실거래·공동주택은 활용신청이 서비스별로 분리 |
| 공동주택 세대수 파싱 None | `complex_profile.json` 필드명이 `households`,`used_date`(정규화됨) — `kaptdaCnt` 아님 | 정규화 필드명 사용 | 스킬 산출 JSON 키를 먼저 확인 |
| 단지명 매칭 실패 다수 | 경매표기 vs k-apt 등록명 상이(공백/하이픈/"아파트"/브랜드명) | 이름 정규화(공백·하이픈·"아파트"·"단지" 제거) 후 부분매칭 | 그래도 실패: **호반베르디움더숲↔목감호반써밋더숲**, **초지역메이저타운푸르지오-에코↔초지역메이저타운-에코단지** 처럼 브랜드명 자체가 달라 자동매칭 한계 |
| 공동주택 API 502 Bad Gateway | 일시적 서버 오류 | 재시도(3~4회) 로직 | 공공 API는 재시도 필수 |
| **금천 24건 vs 내 7건** | ①시/군/구 서버필터 미적용(폼에 코드 있어도 서버가 관할 전체 반환) ②페이지네이션 100건(10페이지) 한도 | 다중 페이지그룹(`nextPage_btn`) 수집 구현 → 100→160건 | **시/군/구 완전성은 근본 한계**. 소규모 정밀검색은 사이트 자체 검색이 정답 |
| 최저가 값 불일치(2.34↔2.93억) | `minmaePrice`(최초 최저가=감정가) vs `notifyMinmaePrice1`(현재 최저) | **`notifyMinmaePrice1` 사용** | 저감 반영된 현재 최저가는 notify 필드 |
| `'list' object has no attribute 'get'` | `감정평가서요약`이 dict 아닌 list | 타입 확인 후 처리 | 스킬 JSON은 섹션별 타입 상이 |
| **브라우저 fetch로 검색 API 재호출 → IP 차단** (2026-07-05 실측) | 스텔스 세션이라도 **UI 클릭 없이 fetch 변조 재전송은 차단** | UI select/input + 검색버튼 클릭 경로로만 | ⚠️ **차단 시 수시간~하루 대기. 같은 IP 재시도 금지** (court-auction-scraper SKILL에 반영됨) |

### 검증된 안티패턴 (하지 말 것)
- courtauction.go.kr에 `requests`/`curl` 직접 POST → IP 차단
- Playwright 기본 headless(스텔스 없이) → WebSquare 렌더링 중단
- 브라우저 컨텍스트에서 `fetch`로 searchControllerMain 재호출 → IP 차단
- 시/군/구 필터를 프로그램적 select로 서버에 걸려는 시도 → 서버 미적용(사후필터로만)
- PublicDataReader 1.0.x 사용

## 4. 재현 가이드

### 전제조건
- **설치**: `pip3 install "PublicDataReader>=1.1.1" python-dotenv playwright --break-system-packages` + `python3 -m playwright install chromium`
- **키**: `/Users/leomyung/auction_harness/.env` 에 `PUBLIC_DATA_SERVICE_KEY=<공공데이터포털 Decoding 키>` (data.go.kr: 아파트 실거래가 + 공동주택 기본정보 + 단지목록 서비스 각각 활용신청)
- **주의**: 경매 스크래핑은 IP 차단 위험 — UI 클릭 경로만, 차단 시 대기

### 단계
```bash
# (1) 물건 수집 — 지역·용도·유찰 필터
cd ~/.claude/skills/court-auction-scraper/scripts
python3 scrape_auction_filtered.py --court 서울남부지방법원 --sido 서울특별시 --sgg 금천구 \
  --lcl 건물 --mcl 주거용건물 --scl 다세대주택 --flbd-min 1회 --max-price 300000000 -o out.csv
#   ※ 시/군/구 완전성 한계 → 소규모 정밀검색은 사이트에서 직접 확인 병행

# (2) 지역 시세(아파트만) — realprice-flow
cd ~/.claude/skills/realprice-flow/scripts
python3 collect.py --district 시흥시 --years 1 --end 202606 --workdir ./out   # sale.csv/rent.csv

# (3) 단지 스펙(세대수·사용승인) — apt-value
cd ~/.claude/skills/apt-value/scripts
python3 fetch_complex.py --name 한라비발디캠퍼스 --district 시흥시 --workdir ./out  # complex_profile.json

# (4) 단일 사건 권리분석 — court-auction-detail
cd ~/.claude/skills/court-auction-detail/scripts
python3 analyze_case.py --court 서울남부지방법원 --case '2025타경12890' -o case.json
#   → 명세서(최선순위=말소기준·임차인·대항력·인수), 현황조사, 인근매각(실제 낙찰가율)
```
- 핵심 경로: 스크래퍼 템플릿 `~/.claude/skills/court-auction-scraper/scripts/templates/*.json`
- 하네스: `~/Users/leomyung/auction_harness/.claude/` (agents + auction-orchestrator + ranker)

## 5. 재발방지 체크리스트
- [ ] PublicDataReader ≥1.1.1 확인
- [ ] `.env`의 `PUBLIC_DATA_SERVICE_KEY` + 공동주택 API 활용신청 완료 확인
- [ ] 경매 최저가는 `notifyMinmaePrice1`, 유찰횟수는 저감율 역산으로 판정
- [ ] 경매 스크래핑: UI 클릭 경로만, fetch 재호출 금지, 차단 시 대기
- [ ] 시/군/구 소규모 정밀검색은 사이트 결과와 대조(스크래퍼 단독 신뢰 금지)
- [ ] 단지 스펙 매칭 실패 시 k-apt 등록명 직접 확인(브랜드명 상이 가능)
- [ ] **다세대/빌라 권리분석 필수**: 대항력 있는 선순위 임차인 보증금 인수 여부 = 생존선. "HUG 특별매각조건"/"대항력 없음(전입>말소기준)" 확인 전엔 저감율(할인)을 믿지 말 것

## 6. 산출물
- **저장소**: https://github.com/earthskyisbig/auction_harness
  - `f514272` 루프엔지니어링 메타스킬 / `b28c332` 경매 하네스 / `7b2cfe7` README / `db76efe` .gitignore
  - `17f0ff0` 필터 크롤러 번들 / `2762555` 강의 커리큘럼 / `e900953` 다중 페이지그룹 수집
- **글로벌 스킬 개선**: `~/.claude/skills/harness`(Phase6/7 루프), `~/.claude/skills/court-auction-scraper`(필터 크롤러+다중그룹)
- **분석 결과**: 안산 아파트 5건(한라비발디·은계브리즈힐 등), 금천 다세대 유찰1회 4건 권리스캔(#10322 함정=보증금2.8억 인수, #12979·#12890 HUG확약 통과, #656 주의)
- **본 기록 스킬**: `~/.claude/skills/myprompt`

## 도메인 교훈 (경매)
- 서울 유찰1회=80%(20%저감), 경기 유찰1회=70%(30%저감).
- "최저가 −30%"는 착시 — 실제 낙찰가는 감정가의 80~90%(인근매각 사례로 확인).
- 다세대는 거의 전세 낀 대항력 임차인 존재 → 인수 리스크가 아파트보다 훨씬 큼. 인근매각 동일단지 낙찰가율이 최고의 시세 신호.
