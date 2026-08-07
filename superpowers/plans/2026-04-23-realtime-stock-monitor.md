# 실시간 주가 모니터 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 맥북에서 1분마다 주가를 수집하여 Gist에 저장하고, GitHub Pages에서 GitHub API(conditional requests)로 ~90초 지연의 실시간 차트를 제공한다.

**Architecture:** Python 데몬이 1분마다 네이버 금융 API로 포트폴리오 종목 가격을 수집하고, Gist API(`PATCH /gists/{id}`)로 업데이트한다. `realtime.html`은 `generate_site.py`에서 생성되어 `stock-dashboard` GitHub Pages에 포함된다. 브라우저는 90초마다 `GET /gists/{id}`를 `If-None-Match` 헤더와 함께 호출하여 CDN 캐시(raw URL의 5분 제약) 없이 데이터를 받는다. Rate limit은 비인증 60/hr 제한이고, 90초 간격 = 40 req/hr로 단일 탭 사용 시 안전. 다중 탭 사용 시 한도 초과 가능(UI에서 경고). 변동 없으면 304 반환받아 불필요한 JSON 파싱을 생략한다 (rate limit은 동일하게 소모되지만 네트워크/CPU 절약).

**Tech Stack:** Python 3 (stdlib only), GitHub Gist API, Chart.js v4 (CDN), macOS launchd

---

## 파일 구조

| 파일 | 역할 | 상태 |
|------|------|------|
| `scripts/realtime_collector.py` | 데몬: 가격 수집 + Gist push | 신규 |
| `scripts/realtime_index.html` | 프론트엔드 HTML 원본 (generate_site.py가 docs/로 복사) | 신규 |
| `tests/test_realtime_collector.py` | 수집기 단위 테스트 | 신규 |
| `scripts/generate_site.py` | realtime.html 복사 + 대시보드 링크 추가 | 수정 |
| `data/intraday/` | 당일 분봉 로컬 백업 | 신규 디렉토리 |
| `~/Library/LaunchAgents/com.stock.realtime.plist` | macOS 자동실행 | 신규 (시스템) |

**Gist (외부):** `data.json` 파일 1개를 담은 public gist — 수집기가 관리.

---

### Task 1: 셋업 — Gist 권한 + Gist 생성

**Files:**
- Create: `data/intraday/.gitkeep`

- [ ] **Step 1: gh 토큰에 gist 권한 추가**

```bash
gh auth refresh --scopes gist
```

브라우저가 열리면 승인. 완료 후 확인:

```bash
gh auth status
```

Expected: Token scopes에 `gist` 포함

- [ ] **Step 2: data.json Gist 생성 (filename=data.json으로 바로)**

프론트엔드가 첫 로드 시 에러 없이 동작하도록 모든 필드 포함:

```bash
cat > /tmp/data.json << 'JSON'
{"updated_at":"초기화 중","market_open":false,"sectors":[],"cash":0,"holdings":[],"intraday":{},"daily":{}}
JSON
gh gist create /tmp/data.json --public --desc "실시간 주가 데이터"
rm /tmp/data.json
```

Expected: Gist URL 출력 (예: `https://gist.github.com/choguitar/abc123...`)

**출력된 URL에서 Gist ID를 기록해둔다** (예: `abc123def456`). Task 2, Task 4에서 사용.

- [ ] **Step 3: Gist API 동작 + CORS 확인**

```bash
GIST_ID="여기에_gist_id"
gh api /gists/$GIST_ID --jq '.files["data.json"].content'
```

Expected: 위에서 넣은 JSON 출력

```bash
# CORS 확인 (브라우저에서 호출 가능한지)
curl -sI -H "Origin: https://choguitar.github.io" "https://api.github.com/gists/$GIST_ID" | grep -i 'access-control-allow-origin'
```

Expected: `access-control-allow-origin: *`

- [ ] **Step 4: 로컬 디렉토리 생성 + 커밋**

```bash
mkdir -p data/intraday
touch data/intraday/.gitkeep
git add data/intraday/.gitkeep
git commit -m "feat: realtime monitor intraday 백업 디렉토리"
```

---

### Task 2: 수집기 핵심 함수 + 테스트

**Files:**
- Create: `scripts/realtime_collector.py`
- Create: `tests/test_realtime_collector.py`

- [ ] **Step 1: 테스트 작성**

```python
# tests/test_realtime_collector.py
import sys
import os

sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "scripts"))

from datetime import datetime, timezone, timedelta
from realtime_collector import is_market_hours, parse_realtime_response, build_data_json

KST = timezone(timedelta(hours=9))


def test_market_hours_weekday_10am():
    assert is_market_hours(datetime(2026, 4, 23, 10, 30, tzinfo=KST)) is True


def test_market_hours_open_900():
    assert is_market_hours(datetime(2026, 4, 23, 9, 0, tzinfo=KST)) is True


def test_market_hours_pre_open_850_excluded():
    # 8:50~9:00 동시호가 — closePrice가 전일 종가라 제외
    assert is_market_hours(datetime(2026, 4, 23, 8, 50, tzinfo=KST)) is False


def test_market_hours_before_open():
    assert is_market_hours(datetime(2026, 4, 23, 8, 0, tzinfo=KST)) is False


def test_market_hours_close_1540():
    assert is_market_hours(datetime(2026, 4, 23, 15, 40, tzinfo=KST)) is True


def test_market_hours_after_close():
    assert is_market_hours(datetime(2026, 4, 23, 16, 0, tzinfo=KST)) is False


def test_market_hours_weekend():
    assert is_market_hours(datetime(2026, 4, 25, 10, 30, tzinfo=KST)) is False


def test_parse_normal():
    raw = {
        "datas": [
            {
                "closePrice": "5,100",
                "fluctuationsRatio": "0.39",
                "accumulatedTradingVolume": "1,336,059",
            }
        ]
    }
    result = parse_realtime_response(raw)
    assert result == {"price": 5100, "change_pct": 0.39, "volume": 1336059}


def test_parse_zero_price():
    assert parse_realtime_response({"datas": [{"closePrice": "0"}]}) is None


def test_parse_empty():
    assert parse_realtime_response({}) is None
    assert parse_realtime_response({"datas": [{}]}) is None
    assert parse_realtime_response({"datas": []}) is None  # 빈 리스트도 None


def test_build_data_json_structure():
    portfolio = {
        "holdings": [
            {
                "code": "001200",
                "name": "유진투자증권",
                "sector": "금융_기타",
                "avg_price": 5458,
                "shares": 180,
            }
        ],
        "cash": 68819,
    }
    intraday = {"001200": [["09:00", 5100], ["09:01", 5120]]}
    daily = {"001200": [["03-25", 5400, 5600, 5350, 5500]]}
    latest = {"001200": {"price": 5120, "change_pct": 0.39, "volume": 1336059}}

    result = build_data_json(portfolio, intraday, daily, latest)

    assert result["cash"] == 68819
    assert result["sectors"] == ["금융_기타"]
    assert len(result["holdings"]) == 1

    h = result["holdings"][0]
    assert h["price"] == 5120
    assert h["change_pct"] == 0.39
    assert h["volume"] == 1336059

    assert result["intraday"] == intraday
    assert result["daily"] == daily


def test_build_data_json_no_latest():
    portfolio = {
        "holdings": [
            {
                "code": "001200",
                "name": "유진투자증권",
                "sector": "금융_기타",
                "avg_price": 5458,
                "shares": 180,
            }
        ],
        "cash": 0,
    }
    result = build_data_json(portfolio, {}, {}, {})
    assert result["holdings"][0]["price"] == 5458  # avg_price fallback
```

- [ ] **Step 2: 구현 — realtime_collector.py 핵심 함수**

```python
# scripts/realtime_collector.py
#!/usr/bin/env python3
"""실시간 주가 수집 데몬 — 1분 수집, 1분 Gist push"""

import json
import time
import urllib.request
from datetime import datetime, timezone, timedelta
from pathlib import Path

KST = timezone(timedelta(hours=9))
ROOT = Path(__file__).resolve().parent.parent
DATA_DIR = ROOT / "data"
INTRADAY_DIR = DATA_DIR / "intraday"
PORTFOLIO_FILE = DATA_DIR / "portfolio.json"
LATEST_PRICES_FILE = DATA_DIR / "latest_prices.json"

# ===== 여기에 Task 1에서 생성한 Gist ID 입력 =====
GIST_ID = ""
# ================================================

UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
NAVER_HEADERS = {"User-Agent": UA, "Referer": "https://finance.naver.com"}
FETCH_INTERVAL = 60  # 1분


def is_market_hours(now: datetime) -> bool:
    """장 시간: 평일 9:00~15:40 (정규거래 + 장마감 동시호가)
    장전 동시호가(8:50~9:00)는 네이버 closePrice가 전일 종가를 반환하므로 제외.
    """
    if now.weekday() >= 5:
        return False
    minutes = now.hour * 60 + now.minute
    return 540 <= minutes <= 940


def safe_int(val) -> int:
    if val is None:
        return 0
    return int(str(val).replace(",", "").strip() or "0")


def parse_realtime_response(data: dict) -> dict | None:
    """네이버 실시간 API 응답 파싱"""
    datas = data.get("datas") or []
    stock = datas[0] if datas else {}
    price = safe_int(stock.get("closePrice"))
    if price == 0:
        return None
    return {
        "price": price,
        "change_pct": float(stock.get("fluctuationsRatio", 0) or 0),
        "volume": safe_int(stock.get("accumulatedTradingVolume")),
    }


def fetch_current_price(code: str) -> dict | None:
    """네이버 실시간 API에서 현재가 조회"""
    url = f"https://polling.finance.naver.com/api/realtime/domestic/stock/{code}"
    try:
        req = urllib.request.Request(url, headers=NAVER_HEADERS)
        with urllib.request.urlopen(req, timeout=10) as resp:
            return parse_realtime_response(json.loads(resp.read()))
    except Exception as e:
        print(f"  [WARN] {code}: {e}")
        return None


def build_data_json(
    portfolio: dict,
    intraday: dict,
    daily: dict,
    latest_prices: dict | None = None,
) -> dict:
    """Gist에 push할 data.json 구성"""
    now = datetime.now(KST)
    latest_prices = latest_prices or {}
    holdings = []
    sectors = set()

    for h in portfolio["holdings"]:
        code = h["code"]
        sectors.add(h["sector"])
        lp = latest_prices.get(code, {})
        price = lp.get("price", h["avg_price"])
        if code in intraday and intraday[code]:
            price = intraday[code][-1][1]

        holdings.append(
            {
                "code": code,
                "name": h["name"],
                "sector": h["sector"],
                "avg_price": h["avg_price"],
                "shares": h["shares"],
                "price": price,
                "change_pct": lp.get("change_pct", 0.0),
                "volume": lp.get("volume", 0),
            }
        )

    return {
        "updated_at": now.strftime("%Y-%m-%d %H:%M:%S"),
        "market_open": is_market_hours(now),
        "sectors": sorted(sectors),
        "cash": portfolio.get("cash", 0),
        "holdings": holdings,
        "intraday": intraday,
        "daily": daily,
    }
```

- [ ] **Step 3: 테스트 실행**

```bash
cd "/Users/mh/Documents/Claude/Projects/주식 관리"
python -m pytest tests/test_realtime_collector.py -v
```

Expected: 14 tests PASS

- [ ] **Step 4: 커밋**

```bash
git add scripts/realtime_collector.py tests/test_realtime_collector.py
git commit -m "feat: realtime collector 핵�� 함수 + 테스트"
```

---

### Task 3: Gist Publisher + 메인 루프

**Files:**
- Modify: `scripts/realtime_collector.py`

- [ ] **Step 1: Gist API 함수 추���**

`scripts/realtime_collector.py` 끝에 추가:

```python
def load_github_token() -> str:
    """gh CLI config에서 GitHub 토큰 읽기"""
    config_path = Path.home() / ".config" / "gh" / "hosts.yml"
    for line in config_path.read_text().split("\n"):
        stripped = line.strip()
        if stripped.startswith("oauth_token:"):
            return stripped.split(":", 1)[1].strip()
    raise ValueError("GitHub token not found in ~/.config/gh/hosts.yml")


def gist_api(method: str, token: str, body: dict | None = None) -> dict:
    """Gist API 호출"""
    url = f"https://api.github.com/gists/{GIST_ID}"
    data = json.dumps(body).encode() if body else None
    req = urllib.request.Request(
        url,
        data=data,
        method=method,
        headers={
            "Authorization": f"token {token}",
            "Accept": "application/vnd.github.v3+json",
            "Content-Type": "application/json",
        },
    )
    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.loads(resp.read())


def push_gist(token: str, content: str) -> str:
    """Gist의 data.json 업데이트, updated_at 반환"""
    body = {"files": {"data.json": {"content": content}}}
    result = gist_api("PATCH", token, body)
    return result.get("updated_at", "")


def save_intraday(intraday: dict, date_str: str):
    """당일 분봉 로컬 백업"""
    INTRADAY_DIR.mkdir(exist_ok=True)
    path = INTRADAY_DIR / f"{date_str}.json"
    with open(path, "w") as f:
        json.dump(intraday, f, ensure_ascii=False)


def load_intraday(date_str: str) -> dict:
    """당일 분봉 로컬 복원 (데몬 재시작 시)"""
    path = INTRADAY_DIR / f"{date_str}.json"
    try:
        with open(path) as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return {}


def load_daily_data(portfolio: dict) -> dict:
    """latest_prices.json에서 포트폴리오 종목의 일별 OHLC 로드"""
    try:
        with open(LATEST_PRICES_FILE) as f:
            prices = json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return {}

    holding_codes = {h["code"] for h in portfolio["holdings"]}
    daily = {}
    for code, stock in prices.get("stocks", {}).items():
        if code not in holding_codes:
            continue
        entries = stock.get("daily", [])
        if entries:
            daily[code] = [
                [
                    d["date"][5:],
                    d.get("open", 0),
                    d.get("high", 0),
                    d.get("low", 0),
                    d["close"],
                ]
                for d in entries
            ]
    return daily
```

- [ ] **Step 2: 메인 루프 추가**

`scripts/realtime_collector.py` 끝에 추가:

```python
def main():
    if not GIST_ID:
        print("ERROR: GIST_ID가 설정되지 않았습니다. scripts/realtime_collector.py 상단을 확인하세요.")
        return

    print("=== 실시간 주가 수집 데몬 시작 ===")
    print(f"Gist ID: {GIST_ID}")

    token = load_github_token()
    with open(PORTFOLIO_FILE) as f:
        portfolio = json.load(f)
    daily = load_daily_data(portfolio)

    today = datetime.now(KST).strftime("%Y-%m-%d")
    intraday = load_intraday(today)
    latest_prices = {}

    while True:
        try:
            now = datetime.now(KST)
            now_date = now.strftime("%Y-%m-%d")

            # 날짜 바뀌면 리셋
            if now_date != today:
                today = now_date
                intraday = {}
                latest_prices = {}
                daily = load_daily_data(portfolio)
                with open(PORTFOLIO_FILE) as f:
                    portfolio = json.load(f)
                print(f"[{today}] 새 거래일 — 데이터 리셋")

            if not is_market_hours(now):
                time.sleep(FETCH_INTERVAL)
                continue

            # 1분마다 가격 수집
            time_str = now.strftime("%H:%M")
            collected = 0

            for h in portfolio["holdings"]:
                code = h["code"]
                result = fetch_current_price(code)
                if result:
                    intraday.setdefault(code, []).append(
                        [time_str, result["price"]]
                    )
                    latest_prices[code] = result
                    collected += 1
                time.sleep(0.3)

            save_intraday(intraday, today)

            # Gist push
            try:
                data = build_data_json(portfolio, intraday, daily, latest_prices)
                content = json.dumps(data, ensure_ascii=False, separators=(",", ":"))
                push_gist(token, content)
                print(f"[{time_str}] {collected}종목 수집 → Gist push 완료")
            except Exception as e:
                print(f"[{time_str}] {collected}종목 수집 → Gist push 실패: {e}")

        except KeyboardInterrupt:
            print("\n데몬 종료")
            break
        except Exception as e:
            print(f"[ERROR] {e}")
            time.sleep(10)

        time.sleep(FETCH_INTERVAL)


if __name__ == "__main__":
    main()
```

- [ ] **Step 3: GIST_ID 설정**

Task 1 Step 2에서 얻은 Gist ID를 `scripts/realtime_collector.py` 상단에 입력:

```python
GIST_ID = "여기에_실제_gist_id"
```

- [ ] **Step 4: 로컬 동작 테스트**

```bash
python scripts/realtime_collector.py
```

장 시간이면:
- `[HH:MM] N종목 수집 → Gist push 완료` 로그 확인
- `gh api /gists/{GIST_ID} --jq '.files["data.json"].content' | python3 -m json.tool | head -5` 로 데이터 확인

장 외 시간이면:
- 로그 없이 대기 (정상). Ctrl+C로 종료.
- `is_market_hours` 조건을 잠시 주석처리하여 push 동작 확인 후 복원.

- [ ] **Step 5: 커밋**

```bash
git add scripts/realtime_collector.py
git commit -m "feat: realtime collector Gist publisher + 메인 루프"
```

---

### Task 4: 프론트엔드 HTML

**Files:**
- Create: `scripts/realtime_index.html`

이 파일은 `generate_site.py`가 `docs/realtime.html`로 복사한다 (Task 5). 브라우저에서 GitHub API(`/gists/{id}`)를 conditional request로 호출하여 데이터를 받고 Chart.js로 렌더링.

- [ ] **Step 1: realtime_index.html 작성**

**중요:** 파일 내 `GIST_ID` 변수에 Task 1에서 생성한 실제 Gist ID를 넣어야 한다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
<link rel="icon" type="image/svg+xml" href="/stock-dashboard/favicon.svg">
<title>실시간 주가 모니터</title>
<style>
:root{--bg:#0f1117;--card:#1a1d27;--border:#2a2d3a;--text:#e1e4ed;--muted:#8b8fa3;--accent:#6c8aff;--green:#22c55e;--red:#ef4444;--orange:#f59e0b}
*{margin:0;padding:0;box-sizing:border-box}
body{background:var(--bg);color:var(--text);font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;line-height:1.6}
a{color:var(--accent);text-decoration:none}
.container{max-width:960px;margin:0 auto;padding:16px}
.header{display:flex;justify-content:space-between;align-items:center;padding:16px 0;border-bottom:1px solid var(--border);margin-bottom:16px}
.header h1{font-size:1.2rem;font-weight:600}
.meta{color:var(--muted);font-size:0.8rem;display:flex;align-items:center;gap:8px}
.dot{display:inline-block;width:8px;height:8px;border-radius:50%}
.dot.open{background:var(--green)}.dot.closed{background:var(--red)}
.rbtn{background:#22252f;border:1px solid var(--border);color:var(--text);padding:4px 10px;border-radius:6px;cursor:pointer;font-size:0.8rem}
.rbtn:hover{background:var(--border)}
.controls{display:flex;gap:6px;flex-wrap:wrap;margin-bottom:16px;align-items:center}
.sep{width:1px;height:24px;background:var(--border);margin:0 6px}
.btn{background:#22252f;border:1px solid var(--border);color:var(--muted);padding:5px 12px;border-radius:6px;cursor:pointer;font-size:0.78rem;transition:all 0.15s}
.btn:hover{background:var(--border);color:var(--text)}
.btn.on{background:var(--accent);color:#fff;border-color:var(--accent)}
.card{background:var(--card);border:1px solid var(--border);border-radius:10px;padding:16px;margin-bottom:16px}
table{width:100%;border-collapse:collapse;font-size:0.8rem}
th{background:#22252f;color:var(--muted);text-align:left;padding:7px 8px;font-weight:500;white-space:nowrap}
td{padding:6px 8px;border-top:1px solid var(--border);white-space:nowrap}
tr:hover td{background:#1f2230}
.pos{color:var(--green);font-weight:600}.neg{color:var(--red);font-weight:600}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(380px,1fr));gap:14px}
.gitem{background:var(--card);border:1px solid var(--border);border-radius:10px;padding:14px}
.gtitle{color:var(--muted);font-size:0.8rem;margin-bottom:8px;font-weight:500}
.loading{text-align:center;color:var(--muted);padding:60px 0;font-size:0.9rem}
.back{color:var(--muted);font-size:0.82rem;display:inline-block;margin-bottom:8px}
.srow{display:flex;gap:20px;margin-bottom:12px;flex-wrap:wrap}
.sitem .label{color:var(--muted);font-size:0.72rem}
.sitem .value{font-size:1.05rem;font-weight:600}
.rate-warn{background:#2a2000;color:var(--orange);padding:8px 12px;border-radius:6px;font-size:0.78rem;margin-bottom:12px;display:none}
@media(max-width:600px){
  .grid{grid-template-columns:1fr}table{font-size:0.72rem}
  td,th{padding:5px}
  .btn{padding:4px 8px;font-size:0.72rem}
  .header h1{font-size:1rem}
}
</style>
</head>
<body>
<div class="container">
  <a class="back" href="index.html">&larr; 대시보드</a>
  <div class="header">
    <h1>실시간 주가 모니터</h1>
    <div class="meta">
      <span id="status"></span>
      <button class="rbtn" onclick="doFetch()">새로고침</button>
    </div>
  </div>
  <div id="rateWarn" class="rate-warn"></div>
  <div class="controls">
    <button class="btn on" data-m="realtime" onclick="setMode('realtime')">실시간</button>
    <button class="btn" data-m="daily" onclick="setMode('daily')">일봉</button>
    <div class="sep"></div>
    <button class="btn on" data-s="all" onclick="setSector('all')">전체</button>
    <span id="sbtns"></span>
  </div>
  <div id="summary" class="card" style="display:none"></div>
  <div id="charts" class="grid"></div>
  <div id="loading" class="loading">데이터 로딩 중...</div>
</div>

<script src="https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js"></script>
<script>
// ===== 여기에 Gist ID 입력 =====
const GIST_ID = '';
// ===============================

const API = 'https://api.github.com/gists/' + GIST_ID;
let DATA = null, etag = null;
let mode = 'realtime', sector = 'all';
let cmap = {};

/* ── Fetch with Conditional Requests ── */
async function doFetch() {
  try {
    const hdrs = {};
    if (etag) hdrs['If-None-Match'] = etag;
    const r = await fetch(API, {headers: hdrs});

    // Rate limit 체크
    const remain = r.headers.get('X-RateLimit-Remaining');
    const reset = r.headers.get('X-RateLimit-Reset');
    if (remain !== null && parseInt(remain) < 10) {
      const resetTime = new Date(parseInt(reset) * 1000).toLocaleTimeString('ko');
      document.getElementById('rateWarn').style.display = 'block';
      document.getElementById('rateWarn').textContent =
        'API 요청 한도 임박 (잔여: ' + remain + '). ' + resetTime + ' 이후 리셋.';
    } else {
      document.getElementById('rateWarn').style.display = 'none';
    }

    if (r.status === 304) return; // 변동 없음 — 파싱/렌더링 생략
    if (r.status === 403) {
      document.getElementById('rateWarn').style.display = 'block';
      document.getElementById('rateWarn').textContent =
        'API 요청 한도 초과. 잠시 후 자동 재시도합니다.';
      return;
    }
    if (!r.ok) throw new Error('HTTP ' + r.status);

    etag = r.headers.get('ETag');
    const gist = await r.json();
    const raw = gist.files['data.json'];
    if (!raw) throw new Error('data.json not found in gist');
    DATA = JSON.parse(raw.content);
    document.getElementById('loading').style.display = 'none';
    buildSectorBtns();
    render();
  } catch (e) {
    if (!DATA) {
      document.getElementById('loading').textContent =
        '데이터를 불러올 수 없습니다. 수집기가 실행 중인지 확인하세요.';
    }
    console.error(e);
  }
}

/* ── Controls ── */
function setMode(m) {
  mode = m;
  document.querySelectorAll('[data-m]').forEach(
    b => b.classList.toggle('on', b.dataset.m === m));
  render();
}
function setSector(s) {
  sector = s;
  document.querySelectorAll('[data-s]').forEach(
    b => b.classList.toggle('on', b.dataset.s === s));
  render();
}
function buildSectorBtns() {
  if (!DATA) return;
  // 섹터 버튼은 매번 재생성되므로 현재 sector 값에 따라 'on' 클래스 유지
  document.getElementById('sbtns').innerHTML = DATA.sectors.map(s =>
    '<button class="btn' + (sector === s ? ' on' : '') + '" data-s="' + s + '" onclick="setSector(\'' + s + '\')">' +
    s.replace(/_/g,' ') + '</button>'
  ).join('');
  // 정적 "전체" 버튼도 동기화
  document.querySelectorAll('[data-s="all"]').forEach(
    b => b.classList.toggle('on', sector === 'all'));
}
function filtered() {
  if (!DATA) return [];
  return sector === 'all' ? DATA.holdings : DATA.holdings.filter(h => h.sector === sector);
}

/* ── Render ── */
function render() {
  if (!DATA) return;
  renderStatus();
  renderSummary();
  renderCharts();
}

/* KST 기준 장 상태 (클라이언트 시계 기준 — Gist의 market_open은 스테일일 수 있음) */
function isKSTMarketOpen() {
  const fmt = new Intl.DateTimeFormat('en-US', {
    timeZone: 'Asia/Seoul', hour12: false,
    weekday: 'short', hour: '2-digit', minute: '2-digit'
  });
  const p = fmt.formatToParts(new Date());
  const wk = p.find(x => x.type === 'weekday').value;
  if (wk === 'Sat' || wk === 'Sun') return false;
  const hr = parseInt(p.find(x => x.type === 'hour').value);
  const mn = parseInt(p.find(x => x.type === 'minute').value);
  const mins = hr * 60 + mn;
  return mins >= 540 && mins <= 940;  // 9:00~15:40 (장전 동시호가 제외)
}

function getKSTDateStr() {
  return new Date().toLocaleDateString('en-CA', { timeZone: 'Asia/Seoul' });
}

function renderStatus() {
  const o = isKSTMarketOpen();
  const dataDate = DATA.updated_at.slice(0, 10);
  const today = getKSTDateStr();
  const stale = dataDate !== today && dataDate !== '초기화 중';

  // 장 중인데 updated_at이 3분 이상 오래되면 collector 중단 의심
  let liveStale = false;
  if (o && !stale && DATA.updated_at !== '초기화 중') {
    const ts = new Date(DATA.updated_at.replace(' ', 'T') + '+09:00').getTime();
    liveStale = Date.now() - ts > 3 * 60 * 1000;
  }

  let label = '';
  if (stale) {
    label = ' <span style="color:var(--orange)">⚠ ' + dataDate + ' 데이터</span>';
  } else if (liveStale) {
    label = ' <span style="color:var(--orange)">⚠ 수집 중단?</span>';
  }
  document.getElementById('status').innerHTML =
    '<span class="dot ' + (o?'open':'closed') + '"></span> ' +
    (o?'장중':'장 마감') + ' | ' + DATA.updated_at.slice(11,16) + label;
}

function renderSummary() {
  const hs = filtered(), fmt = n => n.toLocaleString();
  let tEval=0, tInv=0;
  const rows = hs.map(h => {
    const ev=h.price*h.shares, inv=h.avg_price*h.shares, pnl=ev-inv;
    const pct=(h.price-h.avg_price)/h.avg_price*100;
    tEval+=ev; tInv+=inv;
    const c=pnl>=0?'pos':'neg', cc=h.change_pct>=0?'pos':'neg';
    return '<tr><td><strong>'+h.name+'</strong></td>'+
      '<td>'+fmt(h.price)+'</td>'+
      '<td class="'+cc+'">'+(h.change_pct>=0?'+':'')+h.change_pct.toFixed(2)+'%</td>'+
      '<td>'+h.shares+'</td><td>'+fmt(ev)+'</td>'+
      '<td class="'+c+'">'+(pct>=0?'+':'')+pct.toFixed(1)+'%</td>'+
      '<td class="'+c+'">'+(pnl>=0?'+':'')+fmt(pnl)+'</td></tr>';
  }).join('');
  const tP=tEval-tInv, tp=tInv?tP/tInv*100:0, tc=tP>=0?'pos':'neg';
  const el=document.getElementById('summary');
  el.style.display='block';
  el.innerHTML=
    '<div class="srow">'+
    '<div class="sitem"><div class="label">총 평가</div><div class="value">'+fmt(tEval+(DATA.cash||0))+'</div></div>'+
    '<div class="sitem"><div class="label">수익률</div><div class="value '+tc+'">'+(tp>=0?'+':'')+tp.toFixed(1)+'%</div></div>'+
    '<div class="sitem"><div class="label">손익</div><div class="value '+tc+'">'+(tP>=0?'+':'')+fmt(tP)+'</div></div>'+
    '</div>'+
    '<div style="overflow-x:auto"><table>'+
    '<thead><tr><th>종목</th><th>현재가</th><th>등락</th><th>수량</th><th>평가금</th><th>수익률</th><th>손익</th></tr></thead>'+
    '<tbody>'+rows+'</tbody></table></div>';
}

/* ── Charts ── */
function renderCharts() {
  Object.values(cmap).forEach(c=>c.destroy()); cmap={};
  const hs=filtered(), ct=document.getElementById('charts');
  ct.innerHTML=hs.map(h=>
    '<div class="gitem"><div class="gtitle">'+h.name+' ('+h.code+')</div>'+
    '<canvas id="c_'+h.code+'" height="180"></canvas></div>'
  ).join('');
  hs.forEach(h=>{
    const cv=document.getElementById('c_'+h.code);
    if(!cv)return;
    cmap[h.code]=mode==='realtime'?drawRT(cv,h):drawDaily(cv,h);
  });
}

function bOpts(){
  return{responsive:true,animation:{duration:0},
    plugins:{legend:{display:true,labels:{color:'#8b8fa3',font:{size:10},boxWidth:12}}},
    scales:{
      x:{ticks:{color:'#8b8fa3',font:{size:9},maxRotation:45},grid:{color:'#2a2d3a'}},
      y:{ticks:{color:'#8b8fa3',font:{size:9},callback:v=>v.toLocaleString()},grid:{color:'#2a2d3a'}}
    }};
}

/* ── Realtime Line Chart ── */
function drawRT(cv,h){
  const e=DATA.intraday[h.code]||[];
  return new Chart(cv,{type:'line',
    data:{labels:e.map(x=>x[0]),datasets:[
      {label:h.name,data:e.map(x=>x[1]),borderColor:'#6c8aff',borderWidth:2,pointRadius:1.5,tension:0.2,fill:false},
      {label:'평단 '+h.avg_price.toLocaleString(),data:Array(e.length).fill(h.avg_price),
       borderColor:'#22c55e',borderWidth:1,pointRadius:0,borderDash:[4,4],fill:false}
    ]},options:bOpts()});
}

/* ── Daily OHLC Bar Chart ── */
function calcMA(c,p){return c.map((_,i)=>{if(i<p-1)return null;const s=c.slice(i-p+1,i+1);return Math.round(s.reduce((a,b)=>a+b,0)/p);});}

function drawDaily(cv,h){
  const e=DATA.daily[h.code]||[];
  if(!e.length) return new Chart(cv,{type:'bar',data:{labels:['데이터 없음'],datasets:[]},options:bOpts()});

  const ohlc=e.map(x=>({d:x[0],o:x[1],h:x[2],l:x[3],c:x[4]}));
  const closes=ohlc.map(d=>d.c);
  const ma5=calcMA(closes,5), ma20=calcMA(closes,20);

  const bodies=ohlc.map(d=>[Math.min(d.o,d.c),Math.max(d.o,d.c)]);
  const bg=ohlc.map(d=>d.c>=d.o?'rgba(34,197,94,0.6)':'rgba(239,68,68,0.6)');
  const bd=ohlc.map(d=>d.c>=d.o?'#22c55e':'#ef4444');

  const vH=ohlc.filter(d=>d.h>0);
  const aH=Math.max(...vH.map(d=>d.h)), aL=Math.min(...vH.map(d=>d.l));
  const pad=(aH-aL)*0.08;

  const opts=bOpts();
  opts.scales.y.min=Math.max(0,aL-pad);
  opts.scales.y.max=aH+pad;

  return new Chart(cv,{type:'bar',
    data:{labels:ohlc.map(d=>d.d),datasets:[
      {data:bodies,backgroundColor:bg,borderColor:bd,borderWidth:1,barPercentage:0.5,label:'OHLC',order:3},
      {type:'line',data:ma5,borderColor:'#f59e0b',borderWidth:1.5,pointRadius:0,borderDash:[4,2],label:'5일선',tension:0.3,fill:false,order:1},
      {type:'line',data:ma20,borderColor:'#ef4444',borderWidth:1.5,pointRadius:0,borderDash:[6,3],label:'20일선',tension:0.3,fill:false,order:1},
      {type:'line',data:Array(ohlc.length).fill(h.avg_price),borderColor:'#22c55e',borderWidth:1,pointRadius:0,borderDash:[4,4],label:'평단',fill:false,order:1}
    ]},
    options:opts,
    plugins:[{id:'wicks',afterDatasetsDraw(chart){
      const meta=chart.getDatasetMeta(0),ctx=chart.ctx,ys=chart.scales.y;
      meta.data.forEach((bar,i)=>{
        const d=ohlc[i]; if(!d||!d.h||!bar)return;
        ctx.beginPath();
        ctx.strokeStyle=d.c>=d.o?'#22c55e':'#ef4444';
        ctx.lineWidth=1;
        ctx.moveTo(bar.x,ys.getPixelForValue(d.h));
        ctx.lineTo(bar.x,ys.getPixelForValue(d.l));
        ctx.stroke();
      });
    }}]
  });
}

/* ── Init ── */
doFetch();
setInterval(doFetch, 90000);
</script>
</body>
</html>
```

- [ ] **Step 2: HTML 내 GIST_ID 설정**

`scripts/realtime_index.html` 내의 `GIST_ID` 변수에 Task 1에서 생성한 실제 Gist ID를 입력:

```javascript
const GIST_ID = '여기에_실제_gist_id';
```

- [ ] **Step 3: 로컬에서 HTML 동작 확인**

수집기가 Gist에 데이터를 한 번이라도 push한 상태에서:

```bash
open scripts/realtime_index.html
```

브라우저 개발자 도구(Console)에서 확인:
- CORS 에러 없이 Gist API 호출 성공
- `DATA` 변수에 데이터 로드
- 차트 렌더링 (실시간 모드: 라인, 일봉 모드: OHLC 바)
- 섹터 필터 동작
- 90초 후 자동 갱신 (304면 콘솔에 에러 없음)

- [ ] **Step 4: 커밋**

```bash
git add scripts/realtime_index.html
git commit -m "feat: realtime 프론트엔드 HTML (Gist API + conditional requests)"
```

---

### Task 5: generate_site.py 수정 — realtime.html 포함 + 대시보드 링크

**Files:**
- Modify: `scripts/generate_site.py:506-524`

- [ ] **Step 1: generate_site.py 수정**

`scripts/generate_site.py`의 `generate()` 함수를 수정. 변경 부분 2곳:

**1) header에 실시간 링크 추가 (509-512행 교체):**

기존:
```python
    body = f"""<header>
<h1>주식 포트폴리오 대시보드</h1>
<div class="date">마지막 업데이트: {now}</div>
</header>
```

변경:
```python
    body = f"""<header>
<h1>주식 포트폴리오 대시보드</h1>
<div class="date">마지막 업데이트: {now}
 <a href="realtime.html" target="_blank"
    style="margin-left:12px;padding:4px 10px;background:#22252f;border:1px solid #2a2d3a;border-radius:6px;font-size:0.8rem;color:#6c8aff;">실시간 주가 &nearr;</a>
</div>
</header>
```

**2) generate() 함수 끝에 realtime.html 복사 추가 (523행 `print("Done!")` 직전):**

기존:
```python
    print(f"  index.html (reports: {len(reports)})")
    print("Done!")
```

변경:
```python
    # realtime.html 복사
    realtime_src = ROOT / "scripts" / "realtime_index.html"
    if realtime_src.exists():
        import shutil
        shutil.copy2(realtime_src, DOCS_DIR / "realtime.html")
        print("  realtime.html")

    print(f"  index.html (reports: {len(reports)})")
    print("Done!")
```

- [ ] **Step 2: 사이트 재생성 + 확인**

```bash
python scripts/generate_site.py
```

Expected:
```
  realtime.html
  index.html (reports: NN)
Done!
```

확인:
```bash
grep "실시간 주가" docs/index.html && ls docs/realtime.html
```

Expected: 링크 태그 + `docs/realtime.html` 파일 존재

- [ ] **Step 3: 커밋**

```bash
git add scripts/generate_site.py
git commit -m "feat: 대시보드에 실시간 주가 페이지 링크 + realtime.html 배포 포함"
```

---

### Task 6: LaunchAgent — 자동 실행

**Files:**
- Create: `~/Library/LaunchAgents/com.stock.realtime.plist` (macOS system, git 밖)

- [ ] **Step 1: Python 경로 확인**

LaunchAgent는 PATH가 제한적이므로 `python3`의 절대 경로가 필요. 사용자 환경에 맞는 경로 확인:

```bash
PYTHON_PATH=$(which python3)
echo "$PYTHON_PATH"
# Expected: /opt/homebrew/bin/python3 (Homebrew) 또는 /usr/bin/python3 (macOS 기본)
```

- [ ] **Step 2: plist 작성 (PYTHON_PATH 치환)**

```bash
cat > ~/Library/LaunchAgents/com.stock.realtime.plist << PLIST
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.stock.realtime</string>
    <key>ProgramArguments</key>
    <array>
        <string>${PYTHON_PATH}</string>
        <string>/Users/mh/Documents/Claude/Projects/주식 관리/scripts/realtime_collector.py</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>ThrottleInterval</key>
    <integer>30</integer>
    <key>StandardOutPath</key>
    <string>/tmp/stock-realtime.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/stock-realtime.err</string>
    <key>WorkingDirectory</key>
    <string>/Users/mh/Documents/Claude/Projects/주식 관리</string>
</dict>
</plist>
PLIST
```

**참고:** `ThrottleInterval: 30`은 설정 에러 등으로 즉시 종료 시 재시작 간격을 최소 30초로 제한하여 tight loop 방지.

- [ ] **Step 3: LaunchAgent 등록**

```bash
launchctl load ~/Library/LaunchAgents/com.stock.realtime.plist
```

확인:
```bash
launchctl list | grep com.stock.realtime
```

Expected: PID + `com.stock.realtime` 표시

- [ ] **Step 4: 로그 확인**

```bash
sleep 5 && tail -5 /tmp/stock-realtime.log
```

Expected: 수집 로그 또는 장 외 대기

관리 명령어:
```bash
# 중지
launchctl unload ~/Library/LaunchAgents/com.stock.realtime.plist
# 시작
launchctl load ~/Library/LaunchAgents/com.stock.realtime.plist
# 로그 실시간
tail -f /tmp/stock-realtime.log
```

---

### Task 7: E2E 확인

- [ ] **Step 1: 변경사항 push + 워크플로우 수동 트리거**

```bash
git push
```

**주의:** `stock-automation.yml` 워크플로우는 `data/reports/*.md`, `portfolio.json`, `transactions.json` 변경에만 자동 실행된다. `scripts/` 변경은 수동 트리거 필요:

```bash
gh workflow run "Stock Automation"
```

워크플로우 진행 확인:
```bash
gh run watch
```

Expected:
- `build-site` job이 `generate_site.py` 실행
- `docs/realtime.html` 생성 및 `stock-dashboard`에 force-push 포함

- [ ] **Step 2: 전체 흐름 확인**

1. `https://choguitar.github.io/stock-dashboard/` 접속 → "실시간 주가 ↗" 링크 확인
2. 링크 클릭 → `realtime.html` 로드
3. 장중이면 차트에 데이터 표시, 장 외면 마지막 데이터 표시
4. **실시간 모드**: 분봉 라인 차트 + 평균단가 점선
5. **일봉 모드**: OHLC 바 차트 + 5일선/20일선 + 평균단가
6. **섹터 필터**: 특정 섹터 선택 시 해당 종목만 표시
7. **자동 갱신**: 90초 후 자동 fetch (Console에서 304/200 확인)
8. **Rate limit**: 우측 상단에 잔여 횟수 경고 없는지 확인

- [ ] **Step 3: 재부팅 테스트**

맥북 재시작 후:
```bash
launchctl list | grep com.stock.realtime
tail -5 /tmp/stock-realtime.log
```

Expected: 데몬 자동 실행 + 수집 로그

---

## 알려진 제약 (MVP 범위 밖)

- **공휴일 미감지**: `is_market_hours`는 주말만 체크. 평일 공휴일(근로자의 날, 어린이날 등 연간 ~15일)에 데몬이 수집을 시도하면 네이버 API가 전일 종가를 반환하여 분봉 데이터가 오염된다. 공휴일엔 수동으로 데몬을 중단해야 한다:
  ```bash
  launchctl unload ~/Library/LaunchAgents/com.stock.realtime.plist
  # 다음 거래일에 다시 시작:
  launchctl load ~/Library/LaunchAgents/com.stock.realtime.plist
  ```
  향후 개선: `holidays` 라이브러리 또는 `pykrx.is_business_day()` 연동.

- **첫 iteration 실패 처리**: 데몬 시작 직후 네이버 API 일부 종목 호출이 실패하면 해당 종목의 `price`가 `avg_price`로 fallback되어 수익률 0%처럼 표시된다. 다음 iteration에서 성공하면 정상화.

- **Public Gist**: 데이터가 public gist에 저장되므로 포트폴리오 종목/수량/평균단가가 누구나 읽을 수 있다. 이미 GitHub Pages 대시보드(stock-dashboard)도 공개이므로 일관된 수준. 민감 정보(계좌번호 등)는 포함되지 않음.

- **다중 탭 Rate Limit**: 단일 IP에서 동일 페이지를 여러 탭으로 열면 60/hr 한도 초과 가능. UI의 rate limit 경고 배너가 표시되면 탭 수를 줄여야 한다.

- **OHLC 전부 0 엣지 케이스**: 네이버 API가 high/low를 0으로 반환하는 극히 드문 경우 일봉 차트 y축 범위가 NaN이 되어 렌더링 실패 가능. 실제 서비스 종목에서는 발생하지 않음.
