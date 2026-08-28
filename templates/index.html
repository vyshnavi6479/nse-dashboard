"""
NSE Hourly Dashboard — hostable web app with day-by-day history
====================================================================
Same logic as nse_excel_live_logger.py (correct NSE API, one row per
hour, only updates on real volume/delivery change) — served as a live
webpage instead of an Excel file, with a date picker so past days
stay browsable, not just today.

Data is stored in a local SQLite file (nse_data.db) so it survives
app restarts. A background thread polls NSE every 5 minutes for all 5
stocks, regardless of how many people are viewing the page.

FREE-TIER HOSTING NOTE (read this before deploying):
Free tiers on most hosting platforms (Render, etc.) put the app to
sleep after ~15 minutes with no visitors, and the background poller
stops running while asleep. That means if nobody opens the page for a
while, you'll get gaps in the hourly data for that period — the app
will catch back up once someone visits again (whatever hour is
"current" at that moment starts fresh), but time already passed while
asleep is simply not recorded. This is a free-tier limitation, not a
bug — an always-on paid tier removes it entirely.

Local run:
    pip install flask requests --break-system-packages
    python app.py
    open http://localhost:5000

Deployment: see README.md.
"""

import sqlite3
import threading
import time
from datetime import datetime
from pathlib import Path
from zoneinfo import ZoneInfo

import requests
from flask import Flask, jsonify, render_template, request

app = Flask(__name__)

# All "which hour is this" and "which date is this" decisions must use
# Indian time, not the server's system clock. Render's containers run in
# UTC, and IST is UTC+5:30 — using datetime.now() (naive/UTC) instead of
# datetime.now(IST) caused hour labels to be off by 5-6 hours from real
# NSE market time (e.g. a row would be mislabeled "hour 5" when it was
# really recorded around 10:30 AM IST). Every "now" used for a label or
# for bucketing data by hour/date must go through now_ist().
IST = ZoneInfo("Asia/Kolkata")


def now_ist() -> datetime:
    return datetime.now(IST)


DEFAULT_STOCKS = ["RELIANCE", "TCS", "INFY", "HDFCBANK", "SBIN"]
BASE_URL = "https://www.nseindia.com"
QUOTE_API_PATH = "/api/NextApi/apiClient/GetQuoteApi"
POLL_INTERVAL_SECONDS = 300  # 5 minutes
DB_PATH = Path(__file__).parent / "nse_data.db"

HEADERS = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
        "(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
    ),
    "Accept": "*/*",
    "Accept-Language": "en-US,en;q=0.9",
    "Accept-Encoding": "gzip, deflate",  # deliberately no "br" — see nse_excel_live_logger.py notes
    "Connection": "keep-alive",
    "Sec-Fetch-Dest": "empty",
    "Sec-Fetch-Mode": "cors",
    "Sec-Fetch-Site": "same-origin",
}

TRIGGER_FIELDS = ["total_traded_volume", "quantity_traded", "deliverable_quantity"]

ROW_FIELDS = [
    "hour_open", "hour_close", "last_price", "day_high", "day_low", "prev_close", "vwap",
    "total_traded_volume", "quantity_traded", "deliverable_quantity",
    "buy_quantity", "sell_quantity", "cum_total_volume", "cum_delivery_volume", "last_updated",
]

# One shared connection + lock — traffic here is low (5 symbols every 5
# min, plus occasional page views), so a single connection is simpler
# and safer than juggling a pool for this scale.
_db_lock = threading.Lock()
_conn = sqlite3.connect(DB_PATH, check_same_thread=False)
_conn.row_factory = sqlite3.Row

_status_lock = threading.Lock()
_status: dict[str, dict] = {s: {"last_success": None, "last_error": None} for s in DEFAULT_STOCKS}


def init_db():
    with _db_lock:
        _conn.execute(f"""
            CREATE TABLE IF NOT EXISTS hourly_data (
                symbol TEXT NOT NULL,
                date TEXT NOT NULL,
                hour TEXT NOT NULL,
                {", ".join(f"{f} TEXT" for f in ROW_FIELDS)},
                PRIMARY KEY (symbol, date, hour)
            )
        """)
        _conn.commit()


# --------------------------------------------------------------------------
# NSE session + fetch (same logic as the tested Excel version)
# --------------------------------------------------------------------------

def create_nse_session() -> requests.Session:
    session = requests.Session()
    session.headers.update(HEADERS)
    session.get(BASE_URL, timeout=10)
    time.sleep(1)
    quote_page = f"{BASE_URL}/get-quote/equity/RELIANCE/Reliance-Industries-Limited"
    session.get(quote_page, timeout=10)
    session.headers["Referer"] = quote_page
    time.sleep(1)
    return session


def extract_hour_row(data: dict, existing_hour_row: dict | None) -> dict:
    responses = data.get("equityResponse", [])
    if not responses:
        raise ValueError("'equityResponse' empty/missing — unexpected API shape.")
    eq = responses[0]
    meta = eq.get("metaData", {})
    trade = eq.get("tradeInfo", {})
    order_book = eq.get("orderBook", {})

    last_price = trade.get("lastPrice")
    now = now_ist()

    if existing_hour_row is not None:
        hour_open = existing_hour_row.get("hour_open")
    else:
        hour_open = last_price

    total_traded_volume = trade.get("totalTradedVolume")
    deliverable_quantity = trade.get("deliveryquantity")

    return {
        "hour_open": hour_open,
        "hour_close": last_price,
        "last_price": last_price,
        "day_high": meta.get("dayHigh"),
        "day_low": meta.get("dayLow"),
        "prev_close": meta.get("previousClose"),
        "vwap": meta.get("averagePrice"),
        "total_traded_volume": total_traded_volume,
        "quantity_traded": trade.get("quantitytraded"),
        "deliverable_quantity": deliverable_quantity,
        "buy_quantity": order_book.get("totalBuyQuantity"),
        "sell_quantity": order_book.get("totalSellQuantity"),
        "cum_total_volume": total_traded_volume,
        "cum_delivery_volume": deliverable_quantity,
        "last_updated": now.strftime("%Y-%m-%d %H:%M:%S IST"),
    }


def fetch_symbol(session: requests.Session, symbol: str, existing_hour_row: dict | None) -> dict:
    params = {"functionName": "getSymbolData", "marketType": "N", "series": "EQ", "symbol": symbol}
    resp = session.get(f"{BASE_URL}{QUOTE_API_PATH}", params=params, timeout=15)
    resp.raise_for_status()
    try:
        data = resp.json()
    except ValueError as e:
        preview = resp.content[:150]
        raise ValueError(f"Non-JSON response ({e}). First 150 bytes: {preview!r}") from e
    return extract_hour_row(data, existing_hour_row)


def has_changed(new_row: dict, existing_row: dict | None) -> bool:
    if existing_row is None:
        return True
    return any(str(new_row.get(f)) != str(existing_row.get(f)) for f in TRIGGER_FIELDS)


# --------------------------------------------------------------------------
# Storage
# --------------------------------------------------------------------------

def get_existing_row(symbol: str, date_str: str, hour_str: str) -> dict | None:
    with _db_lock:
        cur = _conn.execute(
            "SELECT * FROM hourly_data WHERE symbol=? AND date=? AND hour=?",
            (symbol, date_str, hour_str),
        )
        row = cur.fetchone()
    return dict(row) if row else None


def upsert_row(symbol: str, date_str: str, hour_str: str, row: dict):
    cols = ["symbol", "date", "hour"] + ROW_FIELDS
    placeholders = ", ".join("?" for _ in cols)
    values = [symbol, date_str, hour_str] + [row.get(f) for f in ROW_FIELDS]
    with _db_lock:
        _conn.execute(
            f"INSERT OR REPLACE INTO hourly_data ({', '.join(cols)}) VALUES ({placeholders})",
            values,
        )
        _conn.commit()


def get_rows_for_date(symbol: str, date_str: str) -> list[dict]:
    with _db_lock:
        cur = _conn.execute(
            "SELECT * FROM hourly_data WHERE symbol=? AND date=? ORDER BY CAST(hour AS INTEGER)",
            (symbol, date_str),
        )
        rows = cur.fetchall()
    return [dict(r) for r in rows]


def get_available_dates() -> list[str]:
    with _db_lock:
        cur = _conn.execute("SELECT DISTINCT date FROM hourly_data ORDER BY date DESC")
        rows = cur.fetchall()
    return [r["date"] for r in rows]


# --------------------------------------------------------------------------
# Background poller
# --------------------------------------------------------------------------

def poll_loop(symbols: list[str]):
    session_holder = [None]

    def get_session():
        if session_holder[0] is None:
            session_holder[0] = create_nse_session()
        return session_holder[0]

    while True:
        now = now_ist()
        date_str = now.strftime("%Y-%m-%d")
        hour_str = str(now.hour)

        for symbol in symbols:
            try:
                session = get_session()
                existing = get_existing_row(symbol, date_str, hour_str)
                row = fetch_symbol(session, symbol, existing)

                if has_changed(row, existing):
                    upsert_row(symbol, date_str, hour_str, row)

                with _status_lock:
                    _status[symbol]["last_success"] = now.strftime("%Y-%m-%d %H:%M:%S IST")
                    _status[symbol]["last_error"] = None

            except requests.HTTPError as e:
                status_code = e.response.status_code if e.response is not None else None
                if status_code in (401, 403):
                    session_holder[0] = None  # force re-warm next time
                with _status_lock:
                    _status[symbol]["last_error"] = str(e)
            except Exception as e:
                with _status_lock:
                    _status[symbol]["last_error"] = str(e)

            time.sleep(1)  # small gap between symbols

        time.sleep(POLL_INTERVAL_SECONDS)


def start_background_poller(symbols: list[str]):
    thread = threading.Thread(target=poll_loop, args=(symbols,), daemon=True)
    thread.start()


init_db()

# Started at import time (not inside `if __name__ == "__main__"`) so this
# also runs correctly under gunicorn/production WSGI servers, which import
# this module directly and never execute the __main__ block.
_poller_started = False
if not _poller_started:
    start_background_poller(DEFAULT_STOCKS)
    _poller_started = True


# --------------------------------------------------------------------------
# Routes
# --------------------------------------------------------------------------

@app.route("/")
def index():
    return render_template("index.html", symbols=DEFAULT_STOCKS)


@app.route("/api/data")
def api_data():
    date_str = request.args.get("date") or now_ist().strftime("%Y-%m-%d")
    result = {}
    with _status_lock:
        status_snapshot = {s: dict(_status[s]) for s in DEFAULT_STOCKS}
    for symbol in DEFAULT_STOCKS:
        rows = get_rows_for_date(symbol, date_str)
        result[symbol] = {
            "rows": rows,
            "last_success": status_snapshot[symbol]["last_success"],
            "last_error": status_snapshot[symbol]["last_error"],
        }
    return jsonify({"date": date_str, "symbols": result})


@app.route("/api/dates")
def api_dates():
    return jsonify({"dates": get_available_dates()})


if __name__ == "__main__":
    port = int(__import__("os").environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port, debug=False)
