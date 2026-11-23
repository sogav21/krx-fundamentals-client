# 사용 가이드

처음이라면 [README](../README.md) 부터.

<!-- toc -->
- [더 쓰기](#더-쓰기)
- [구조](#구조)
- [개발](#개발)
<!-- /toc -->

## 더 쓰기

여러 종목의 재무제표를 한 번에 (DART `fnlttMultiAcnt`, 최대 100종목/호출):

```python
financials = await dart.fetch_financials_batch(["005930", "000660", "035420"], year=2025)
for ticker, fs in financials.items():
    if fs is not None:
        print(ticker, fs.revenue, fs.net_income)
```

키 여러 개를 순환할 때는 `DartQuotaExceededError` 를 잡아 다음 키로 넘어간다:

```python
from krx_fundamentals_client import DartQuotaExceededError, DartScraper

async def fetch_with_rotation(tickers, keys):
    for key in keys:
        scraper = DartScraper(api_key=key)
        try:
            return [await scraper.fetch_company(t) for t in tickers]
        except DartQuotaExceededError:
            continue
        finally:
            await scraper.close()
    raise RuntimeError("모든 키가 일한도를 소진했다")
```

스크리닝·랭킹 — `NaverScraper` 가 채운 PER·PBR 로 그 자리에서 거르고 정렬한다(KRX 벌크 경로는 막혀 있으니 종목을 넘겨 모은다):

```python
from krx_fundamentals_client import NaverScraper, RankingMetric, rank_stocks, screen_stocks

naver = NaverScraper()
try:
    ratios = await naver.fetch_batch(["005930", "000660", "035420", "005380"])
    value_stocks = screen_stocks(ratios, per_max=15.0, pbr_max=1.2)
    top_by_pbr = rank_stocks(ratios, RankingMetric.PBR)
finally:
    await naver.close()
```

## 구조

```mermaid
flowchart LR
    Caller["호출자 코드"]

    subgraph Client["krx_fundamentals_client"]
        Dart["DartScraper\n(dart.py)"]
        Krx["KrxScraper\n(krx.py)"]
        Naver["NaverScraper\n(naver.py)"]
        Screen["screen_stocks / rank_stocks\n(screening.py, 순수 함수)"]
        Models["Pydantic 모델\n(models/schemas.py)"]
    end

    Dart -->|"corp_code 캐시(24h)"| DartAPI["DART OpenAPI\n기업개황·재무제표·배당·대주주·임원·발행주식수"]
    Krx -->|"OTP 2단계 다운로드"| KrxSite["KRX 정보데이터시스템\nPER·PBR·시가총액·섹터\n(⛔ 현재 로그인 차단)"]
    Naver --> NaverSite["네이버 금융 모바일 API\n시세·PER·PBR·EPS·BPS·배당수익률·시가총액"]

    Caller --> Dart
    Caller --> Krx
    Caller --> Naver
    Caller --> Screen

    Krx -->|"list[InvestmentRatio]"| Screen
    Naver -->|"list[InvestmentRatio]"| Screen
    Screen -->|"필터링/정렬 결과"| Caller

    Dart --> Models
    Krx --> Models
    Naver --> Models
    Models --> Caller
```

DART 상세 조회는 모두 `corp_code` 가 필요하다. 첫 호출에서 전체 매핑을 한 번 내려받아 인스턴스에 24시간 캐시하므로, 여러 종목을 볼 때는 스크레이퍼 인스턴스를 재사용한다.

## 개발

```bash
uv sync --extra dev
uv run pytest                        # 테스트 (HTTP 는 pytest-httpx 목)
uv run ruff check src/ tests/        # 린트
```
