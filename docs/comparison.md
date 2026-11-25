# 다른 방법과 비교

국내 기업 펀더멘탈을 파이썬에서 받는 흔한 방법 셋(OpenDART 직접 호출, pykrx, FinanceDataReader)과 이 라이브러리를 비교한다.
아래 표의 동작은 **2026-11-26 에 직접 실행해 확인한 것**이다(pykrx 1.2.9, finance-datareader 0.9.202, 이 레포 main).
사이트 쪽 사정이 바뀌면 결과도 바뀔 수 있다.

## 한눈에

| | 이 라이브러리 | OpenDART 직접 호출 | pykrx | FinanceDataReader |
|---|---|---|---|---|
| 재무제표(매출·영업이익·순이익·자산·부채·자본) | ✅ DART, Pydantic 모델 | ✅ 원본 JSON | ❌ | △ 네이버(wisereport) 요약표 스크랩. 실측 시 연결 끊김 |
| 전년 대비 증감률 | ✅ `*_prior`·`*_yoy` 필드 | 직접 계산 (`frmtrm_amount`) | ❌ | ❌ |
| 배당·발행주식수·대주주·임원 | ✅ DART | ✅ 원본 JSON | DPS·배당수익률만 (KRX 로그인 필요) | ❌ |
| PER·PBR·EPS·BPS | ✅ 네이버 (종목 단위) | ❌ | KRX 로그인 필요 | ❌ (`StockListing` 에 없음) |
| 애널리스트 컨센서스(목표주가·추정 EPS) | ✅ 네이버 | ❌ | ❌ | ❌ |
| 시세 이력(OHLCV) | ❌ | ❌ | ✅ (로그인 없이 받아짐) | ✅ |
| 전 종목 목록·시가총액 | ❌ (KRX 경로 막힘) | 기업 목록(corpCode.xml)만 | KRX 로그인 필요 | ✅ `StockListing('KRX')` |
| 반환 형태 | Pydantic 모델, `async` | dict | pandas DataFrame, 동기 | pandas DataFrame, 동기 |
| 필요한 것 | DART API 키(무료) | DART API 키(무료) | KRX 계정(`KRX_ID`·`KRX_PW`) | 없음 |

요약하면 이 라이브러리는 **재무제표·배당·주주 같은 공시 숫자와 투자지표를 한 스키마로** 받는 쪽이고,
pykrx·FinanceDataReader 는 **시세와 종목 목록**에 강하다. 둘을 같이 쓰는 경우가 많다.

## OpenDART 직접 호출

OpenDART 는 원본이다. 이 라이브러리의 `DartScraper` 도 결국 같은 엔드포인트(`fnlttSinglAcnt`, `fnlttMultiAcnt`, `alotMatter` 등)를 부른다.
직접 호출하면 다음을 손으로 처리해야 한다. 모두 실제 응답으로 확인했다.

- **종목코드로 못 부른다.** `stock_code=005930` 으로 부르면 `status=100`("필수값(corp_code,bsns_year,reprt_code)이 누락")이 돌아온다.
  먼저 `corpCode.xml` ZIP 을 받아 종목코드 → `corp_code` 매핑을 만들어야 한다. `DartScraper` 는 이 매핑을 인스턴스에 24시간 캐시한다.
- **금액이 쉼표 문자열이다.** 예: `"thstrm_amount": "333,605,938,000,000"`. 숫자로 바꾸는 건 호출자 몫이다.
- **연결(CFS)과 개별(OFS)이 한 응답에 섞여 온다.** 같은 계정이 두 벌 온다. `DartScraper` 는 연결을 우선하고 없을 때만 개별을 쓴다.
- **계정은 `account_nm`(한글 계정명)으로 찾아야 한다.** `DartScraper` 는 주요 계정 6개를 모델 필드로 옮긴다.
- **오류가 HTTP 200 + `status` 코드로 온다.** 공시 없음(`013`)과 일한도 소진(`020`)을 구분하지 않으면 "데이터 없음"과 "키 소진"이 섞인다.
  `DartScraper` 는 `020` 을 `DartQuotaExceededError` 로 던지고, 나머지 코드는 `on_status` 콜백으로 넘긴다.

**직접 호출이 나은 경우:** 주요 6개 계정 밖의 세부 계정(`fnlttSinglAcntAll` 전체 재무제표), 공시 원문, 이 라이브러리가 감싸지 않은 엔드포인트가 필요할 때.

## pykrx

[pykrx](https://github.com/sharebook-kr/pykrx) 는 KRX 정보데이터시스템과 네이버에서 시세·지수·투자지표를 DataFrame 으로 받는다.

- 1.2.9 는 KRX 데이터를 받을 때 로그인한다(환경변수 `KRX_ID`·`KRX_PW`). 없이 `get_market_fundamental("20261120", "20261125", "005930")` 을 부르면
  "KRX 로그인 실패" 메시지와 함께 **빈 DataFrame** 이 돌아왔다. 계정이 있으면 BPS·PER·PBR·EPS·DIV·DPS 를 날짜별로 받는다.
- 같은 조건에서 `get_market_ohlcv` 는 로그인 없이 시세를 돌려줬다.
- 재무제표(매출·이익·자산·부채)는 다루지 않는다.

**pykrx 가 나은 경우:** 일별 시세·지수, KRX 계정이 있을 때 날짜별 PER·PBR 시계열.

## FinanceDataReader

[FinanceDataReader](https://github.com/FinanceData/FinanceDataReader) 는 국내외 시세·종목 목록·경제지표를 DataFrame 으로 받는다.

- `fdr.StockListing('KRX')` 는 2,879개 종목을 돌려줬다. 컬럼은 `Code`·`Name`·`Market`·`Close`·`Marcap`·`Stocks` 등이고 PER·PBR 은 없다.
- `fdr.DataReader('005930', ...)` 는 OHLCV 를 돌려줬다.
- `fdr.SnapDataReader('NAVER/FINSTATE/005930')` 는 네이버 금융(wisereport) 요약 재무제표를 긁는 경로인데,
  2026-11-26 에 두 번 시도했을 때 모두 `ConnectionError (Remote end closed connection without response)` 로 끝났다.
  DART 공시가 아니라 화면에 보이는 요약표라는 점도 다르다.

**FinanceDataReader 가 나은 경우:** 시세 이력, 전 종목 목록·시가총액, 해외 시세·지수.

## 이 라이브러리의 한계

- 시세 이력과 전 종목 목록은 없다. KRX 벌크 경로(`KrxScraper`)는 로그인 때문에 **예외 없이 빈 리스트**를 돌려준다([krx-blocked.md](krx-blocked.md)).
- 네이버 투자지표는 ROE·부채비율 같은 재무비율을 주지 않아 `None` 이다. 재무비율이 필요하면 DART 재무제표로 직접 계산한다.
- DART 재무제표는 주요 계정 6개와 전년 대비 증감만 담는다.
- `async` 전용이다. 동기 코드에서는 `asyncio.run(...)` 으로 감싼다.
