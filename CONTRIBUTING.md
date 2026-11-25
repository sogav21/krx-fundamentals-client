# 기여 가이드

krx-fundamentals-client 에 관심 가져 주셔서 감사합니다. 버그 제보, 문서 수정, 코드 기여를 모두 받습니다.
처음이라면 [good first issue](https://github.com/younghwan91/krx-fundamentals-client/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22) 목록에서 시작하세요.

> English: PRs and issues in English are welcome. Short version: `uv run --extra dev pytest -q` and
> `uv run --extra dev ruff check src tests` must pass; use English Conventional Commits; open a PR from a branch.

## 개발 환경

Python 3.11 이상과 [uv](https://docs.astral.sh/uv/) 를 권장합니다.

```bash
git clone https://github.com/younghwan91/krx-fundamentals-client.git
cd krx-fundamentals-client
uv sync --extra dev
```

uv 없이는 `python -m venv .venv && . .venv/bin/activate && pip install -e ".[dev]"` 입니다.

## 테스트와 린트

```bash
uv run pytest -q                  # 수 초. HTTP 는 pytest-httpx 목
uv run ruff check src tests       # CI 와 같은 ruff==0.16.2 (핀은 의도입니다)
uv lock --check                   # uv.lock 은 커밋합니다
```

- 테스트는 DART·네이버 서버에 실제 요청을 보내지 않습니다. 새 테스트도 pytest-httpx 목으로 써 주세요.
- 라이브러리는 환경변수를 읽지 않습니다(`DartScraper(api_key=...)`). `examples/` 만 `DART_API_KEY` 를 읽습니다. DART 키를 커밋·이슈·PR 에 넣지 마세요.
- `KrxScraper` 는 KRX 가 막아 예외 없이 빈 리스트를 돌려줍니다([docs/krx-blocked.md](docs/krx-blocked.md)). 빈 결과를 "종목 없음" 으로 읽지 마세요.
- `ruff format` 은 CI 가 돌리지 않습니다. 기능 PR 에서 레포 전체를 재포매팅하지 마세요.

## 커밋과 PR

1. `main` 에서 브랜치를 만듭니다(`feat/…`, `fix/…`, `docs/…`).
2. 커밋 메시지는 **영어 [Conventional Commits](https://www.conventionalcommits.org/)** 입니다(`fix(dart): handle quota error`). 본문에는 왜 바꿨는지 적어 주세요.
3. 새 동작에는 테스트를 붙이고 위 명령이 통과하는지 확인한 뒤 PR 을 엽니다. 작고 한 가지 일만 하는 PR 이 빨리 머지됩니다.
4. CI(ruff·pytest 3.11~3.13)가 통과해야 머지됩니다. 버전과 릴리스 태그는 메인테이너가 정합니다.
5. `DartScraper`·`FinancialStatement` 등 공개 API 의 시그니처·필드는 소비자가 있어 바꾸지 않습니다. 필요하면 새 옵션으로 더해 주세요.

## 이슈

- 버그·기능 제안: [Issues](https://github.com/younghwan91/krx-fundamentals-client/issues)
- good first issue: [목록 보기](https://github.com/younghwan91/krx-fundamentals-client/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22)
