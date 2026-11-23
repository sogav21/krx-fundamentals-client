<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/images/brand/wordmark-dark.svg">
  <img alt="krx-fundamentals-client" src="docs/images/brand/wordmark-light.svg" height="52">
</picture>

# krx-fundamentals-client

**국내 기업 펀더멘탈을 DART·KRX·네이버에서 모아 Pydantic 모델로 돌려주는 Python 라이브러리**

서버·캐시·스케줄러 없이 pip 한 줄로 설치해 호출할 때마다 소스에 직접 요청합니다.

[![CI](https://github.com/younghwan91/krx-fundamentals-client/actions/workflows/ci.yml/badge.svg)](https://github.com/younghwan91/krx-fundamentals-client/actions/workflows/ci.yml)
[![Python](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/downloads/)
[![License](https://img.shields.io/github/license/younghwan91/krx-fundamentals-client)](LICENSE)

[사용 가이드](docs/guide.md) · [아키텍처](docs/architecture.md) · [예제](examples/) · [English](README.en.md)

</div>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/images/hero-dark.png">
  <img alt="삼성전자 한 종목의 DART 연결 재무제표 8개년(매출·영업이익)·배당과 네이버 PER·PBR·배당수익률을 라이브러리로 받아 한 스키마로 그린 그림" src="docs/images/hero-light.png" width="100%">
</picture>

<p align="center"><sub>삼성전자 FY2018~FY2025 연결 재무제표(<code>fetch_financials</code>), 주당배당금(<code>fetch_dividends</code>), 네이버 투자지표(<code>NaverScraper.fetch_stock_info</code>) 실제 호출 결과입니다 (<a href="docs/images/hero-data.json">hero-data.json</a>).</sub></p>

## 빠른 시작

PyPI에는 아직 올라와 있지 않아 git에서 바로 설치합니다.

```bash
pip install git+https://github.com/younghwan91/krx-fundamentals-client
```

[opendart.fss.or.kr](https://opendart.fss.or.kr)에서 무료 DART API 키를 발급받아 삼성전자(`005930`) 8개년 재무제표를 조회합니다.

```python
import asyncio
from krx_fundamentals_client import DartScraper

async def main():
    dart = DartScraper(api_key="...")
    try:
        company = await dart.fetch_company("005930")
        print(company.name, company.ceo)
        for year in range(2018, 2026):
            fs = await dart.fetch_financials("005930", year)   # 사업보고서, 연결 우선(없으면 개별)
            if fs:
                print(year, f"매출 {fs.revenue:,.0f}", f"영업이익 {fs.operating_income:,.0f}")
    finally:
        await dart.close()

asyncio.run(main())
```

실행하면 기업명·대표자명과 함께 연도별 매출·영업이익을 출력합니다. 다른 예제는 [`examples/`](examples/)에 있습니다.

## 무엇을 하나

| 소스 | 클래스 | 돌려주는 것 | 수집 단위 | 인증 |
|-----|-------|-----------|----------|------|
| [DART OpenAPI](https://opendart.fss.or.kr) | `DartScraper` | 기업개황, 재무제표(연간·반기·분기), 발행주식수, 배당, 대주주, 임원 | 종목 단위. 재무제표는 `fetch_financials_batch`로 최대 100종목/호출 | API 키 (무료) |
| [네이버 금융](https://m.stock.naver.com) | `NaverScraper` | `InvestmentRatio` 한 건: 현재가·시가총액·PER·PBR·EPS·BPS·배당수익률·거래량·52주 고저·외국인소진율 (2026-10 실측). ROE·부채비율 같은 재무비율은 주지 않아 `None` | 종목 단위, `fetch_batch`로 여러 종목 | 불필요 |
| [KRX 정보데이터시스템](http://data.krx.co.kr) | `KrxScraper` | PER·PBR·시가총액·섹터 | 시장 전체 벌크 CSV | 불필요 (현재 차단) |

- **단일 Pydantic 모델:** `Company`, `FinancialStatement`, `SharesOutstanding`, `InvestmentRatio`, `Dividend`, `Shareholder`, `Executive`, `SectorOverview`를 `krx_fundamentals_client`에서 바로 import합니다([`models/schemas.py`](src/krx_fundamentals_client/models/schemas.py)).
- **스크리닝·랭킹:** `screen_stocks`와 `rank_stocks`(`RankingMetric`)는 `list[InvestmentRatio]`를 받아 그 자리에서 거르고 정렬하는 순수 함수입니다.
- **서버 없는 경량 클라이언트:** 의존성은 `httpx`, `beautifulsoup4`, `lxml`, `pydantic` 넷뿐이며 상태와 저장은 호출자가 맡습니다.

## 왜 이렇게 만들었나

- **서버가 아닌 라이브러리:** FastAPI·캐시·스케줄러를 걷어내고 라이브러리로 바꿨습니다([kiwoom-client](https://github.com/younghwan91/kiwoom-client), [krx-news-client](https://github.com/younghwan91/krx-news-client)와 같은 구조입니다).
- **실패를 구분하는 설계:** DART 일한도 소진은 `DartQuotaExceededError` 예외로, 공시 없음은 `None`으로 돌려줍니다. `on_status` 콜백을 넘기면 반환 타입을 바꾸지 않고 DART status 코드(013, 010, 800, 900)를 받아볼 수 있습니다.
- **막힌 경로의 투명한 공개:** KRX 경로가 조용히 빈 리스트를 내는 현상을 숨기지 않고 [docs/krx-blocked.md](docs/krx-blocked.md)에 코드 추적까지 기록했습니다.

## 주의할 점

- **KRX 경로 차단(2026-08 실측):** KRX가 로그인을 걸어 `KrxScraper`가 예외 없이 빈 리스트를 돌려줍니다. 시세·PER·PBR 같은 종목별 투자지표는 `NaverScraper`를 사용하세요([docs/krx-blocked.md](docs/krx-blocked.md)).
- **누락 수치 처리:** 소스가 항목을 주지 않으면 값을 지어내지 않고 `None`으로 비워 둡니다.
- **DART `corp_code` 캐시:** 첫 호출에서 전체 매핑을 내려받아 인스턴스에 24시간 캐시하므로 여러 종목을 조회할 때는 인스턴스를 재사용하세요.

## 문서

| 문서 | 내용 |
|---|---|
| [docs/guide.md](docs/guide.md) | 배치 조회, 키 순환, 스크리닝·랭킹, 구조 다이어그램, 개발 명령 |
| [docs/architecture.md](docs/architecture.md) | 스크레이퍼 구조, 랭킹·스크리닝, DART 호출 순서 |
| [docs/krx-blocked.md](docs/krx-blocked.md) | KRX 차단 코드 추적 |
| [`examples/`](examples/) | 기본 사용·스크리닝·포트폴리오 예제 |

## 기여 및 문의

- 버그 제보, 질문, 기능 제안은 [GitHub Issues](https://github.com/younghwan91/krx-fundamentals-client/issues)에 남겨주세요.
- 만든 사람: **채영환 (Younghwan Chae)** · [GitHub @younghwan91](https://github.com/younghwan91) · [LinkedIn](https://www.linkedin.com/in/younghwan-chae/)

## 라이선스

[Apache-2.0](LICENSE)
