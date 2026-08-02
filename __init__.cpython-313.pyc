import asyncio
import base64
import json
import math
import os
import smtplib
import ssl
import threading
from datetime import datetime, timezone
from email.message import EmailMessage
from pathlib import Path
from typing import Any
from zoneinfo import ZoneInfo

import httpx
from fastapi import Depends, FastAPI, HTTPException, Request, status
from fastapi.responses import HTMLResponse, JSONResponse
from fastapi.security import HTTPBasic, HTTPBasicCredentials

APP_NAME = "Vincent's Bitcoin Paper Bot"
PAIR = os.getenv("KRAKEN_PAIR", "XBTGBP")
DISPLAY_PAIR = os.getenv("DISPLAY_PAIR", "BTC/GBP")
INTERVAL_MIN = int(os.getenv("INTERVAL_MIN", "5"))
CHECK_SECONDS = int(os.getenv("CHECK_SECONDS", "300"))
STARTING_CASH = float(os.getenv("STARTING_CASH", "1000"))
POSITION_PCT = float(os.getenv("POSITION_PCT", "0.20"))
TAKE_PROFIT_PCT = float(os.getenv("TAKE_PROFIT_PCT", "0.025"))
STOP_LOSS_PCT = float(os.getenv("STOP_LOSS_PCT", "0.015"))
TRAILING_STOP_PCT = float(os.getenv("TRAILING_STOP_PCT", "0.012"))
FEE_RATE = float(os.getenv("FEE_RATE", "0.004"))
DASHBOARD_PASSWORD = os.getenv("DASHBOARD_PASSWORD", "change-me")
STATE_PATH = Path(os.getenv("STATE_PATH", "/app/data/state.json"))
LOCAL_TZ = ZoneInfo(os.getenv("LOCAL_TIMEZONE", "Europe/London"))
DAILY_EMAIL_HOUR = int(os.getenv("DAILY_EMAIL_HOUR_LOCAL", "8"))

security = HTTPBasic()
app = FastAPI(title=APP_NAME)
state_lock = threading.RLock()
loop_task: asyncio.Task | None = None


def now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


def default_state() -> dict[str, Any]:
    return {
        "created_at": now_iso(),
        "updated_at": now_iso(),
        "running": True,
        "cash_gbp": STARTING_CASH,
        "btc": 0.0,
        "entry_price": None,
        "position_cost": 0.0,
        "highest_price": None,
        "position_opened_at": None,
        "last_price": None,
        "last_signal": "WAITING FOR MARKET DATA",
        "last_reason": "The bot has not completed its first market check yet.",
        "last_check_at": None,
        "last_error": None,
        "indicators": {},
        "trades": [],
        "events": [],
        "last_daily_email_date": None,
    }


def load_state() -> dict[str, Any]:
    STATE_PATH.parent.mkdir(parents=True, exist_ok=True)
    if not STATE_PATH.exists():
        state = default_state()
        save_state(state)
        return state
    try:
        return json.loads(STATE_PATH.read_text())
    except Exception:
        backup = STATE_PATH.with_suffix(".broken.json")
        try:
            STATE_PATH.replace(backup)
        except Exception:
            pass
        state = default_state()
        save_state(state)
        return state


def save_state(state: dict[str, Any]) -> None:
    state["updated_at"] = now_iso()
    STATE_PATH.parent.mkdir(parents=True, exist_ok=True)
    temp = STATE_PATH.with_suffix(".tmp")
    temp.write_text(json.dumps(state, indent=2))
    temp.replace(STATE_PATH)


state = load_state()


def authenticate(credentials: HTTPBasicCredentials = Depends(security)) -> str:
    expected_user = os.getenv("DASHBOARD_USER", "vincent")
    user_ok = secrets_compare(credentials.username, expected_user)
    pass_ok = secrets_compare(credentials.password, DASHBOARD_PASSWORD)
    if not (user_ok and pass_ok):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Basic"},
        )
    return credentials.username


def secrets_compare(a: str, b: str) -> bool:
    import secrets
    return secrets.compare_digest(a.encode(), b.encode())


def ema(values: list[float], period: int) -> float:
    if len(values) < period:
        raise ValueError("Not enough candles")
    alpha = 2 / (period + 1)
    result = sum(values[:period]) / period
    for value in values[period:]:
        result = alpha * value + (1 - alpha) * result
    return result


def rsi(values: list[float], period: int = 14) -> float:
    if len(values) <= period:
        raise ValueError("Not enough candles")
    gains: list[float] = []
    losses: list[float] = []
    for previous, current in zip(values[:-1], values[1:]):
        change = current - previous
        gains.append(max(change, 0.0))
        losses.append(max(-change, 0.0))
    avg_gain = sum(gains[-period:]) / period
    avg_loss = sum(losses[-period:]) / period
    if avg_loss == 0:
        return 100.0
    rs = avg_gain / avg_loss
    return 100 - (100 / (1 + rs))


async def fetch_market() -> tuple[list[dict[str, float]], float]:
    url = "https://api.kraken.com/0/public/OHLC"
    params = {"pair": PAIR, "interval": INTERVAL_MIN}
    timeout = httpx.Timeout(15.0)
    async with httpx.AsyncClient(timeout=timeout, headers={"User-Agent": "VincentPaperBot/1.0"}) as client:
        response = await client.get(url, params=params)
        response.raise_for_status()
        payload = response.json()
    if payload.get("error"):
        raise RuntimeError("Kraken API: " + ", ".join(payload["error"]))
    result = payload.get("result", {})
    key = next((k for k in result.keys() if k != "last"), None)
    if not key:
        raise RuntimeError("Kraken returned no candle data")
    rows = result[key]
    candles = []
    for row in rows:
        candles.append({
            "time": float(row[0]),
            "open": float(row[1]),
            "high": float(row[2]),
            "low": float(row[3]),
            "close": float(row[4]),
            "vwap": float(row[5]),
            "volume": float(row[6]),
        })
    if len(candles) < 60:
        raise RuntimeError("Not enough Kraken candles")
    return candles, candles[-1]["close"]


def email_enabled() -> bool:
    return all(os.getenv(k) for k in ["SMTP_HOST", "SMTP_USER", "SMTP_PASSWORD", "EMAIL_TO"])


def send_email(subject: str, body: str) -> bool:
    if not email_enabled():
        return False
    host = os.environ["SMTP_HOST"]
    port = int(os.getenv("SMTP_PORT", "465"))
    user = os.environ["SMTP_USER"]
    password = os.environ["SMTP_PASSWORD"]
    email_to = os.environ["EMAIL_TO"]
    email_from = os.getenv("EMAIL_FROM", user)
    mode = os.getenv("SMTP_MODE", "ssl").lower()

    msg = EmailMessage()
    msg["From"] = email_from
    msg["To"] = email_to
    msg["Subject"] = subject
    msg.set_content(body)

    context = ssl.create_default_context()
    if mode == "starttls":
        with smtplib.SMTP(host, port, timeout=20) as smtp:
            smtp.starttls(context=context)
            smtp.login(user, password)
            smtp.send_message(msg)
    else:
        with smtplib.SMTP_SSL(host, port, context=context, timeout=20) as smtp:
            smtp.login(user, password)
            smtp.send_message(msg)
    return True


async def send_email_async(subject: str, body: str) -> bool:
    return await asyncio.to_thread(send_email, subject, body)


def portfolio_snapshot(s: dict[str, Any], price: float | None = None) -> dict[str, float]:
    p = price if price is not None else (s.get("last_price") or 0.0)
    equity = s["cash_gbp"] + s["btc"] * p
    pnl = equity - STARTING_CASH
    pnl_pct = pnl / STARTING_CASH if STARTING_CASH else 0.0
    return {"equity_gbp": equity, "pnl_gbp": pnl, "pnl_pct": pnl_pct}


def record_event(kind: str, message: str) -> None:
    state["events"].insert(0, {"time": now_iso(), "kind": kind, "message": message})
    state["events"] = state["events"][:100]


async def buy(price: float, reason: str) -> None:
    equity = portfolio_snapshot(state, price)["equity_gbp"]
    allocation = min(state["cash_gbp"], equity * POSITION_PCT)
    if allocation < 5:
        return
    fee = allocation * FEE_RATE
    net = allocation - fee
    btc = net / price
    state["cash_gbp"] -= allocation
    state["btc"] = btc
    state["entry_price"] = price
    state["position_cost"] = allocation
    state["highest_price"] = price
    state["position_opened_at"] = now_iso()
    state["last_signal"] = "PAPER BUY"
    state["last_reason"] = reason
    record_event("BUY", f"Simulated BUY £{allocation:.2f} of BTC at £{price:,.2f}. {reason}")
    save_state(state)
    snap = portfolio_snapshot(state, price)
    await send_email_async(
        f"PAPER BUY — BTC at £{price:,.2f}",
        f"Vincent's Bitcoin Paper Bot simulated a BUY.\n\n"
        f"Pair: {DISPLAY_PAIR}\nPrice: £{price:,.2f}\nAllocated: £{allocation:.2f}\n"
        f"Fee assumption: £{fee:.2f}\nReason: {reason}\n"
        f"Virtual equity: £{snap['equity_gbp']:.2f}\n\nNo real money was used.",
    )


async def sell(price: float, reason: str) -> None:
    if state["btc"] <= 0:
        return
    btc = state["btc"]
    gross = btc * price
    fee = gross * FEE_RATE
    proceeds = gross - fee
    cost = state.get("position_cost", 0.0)
    pnl = proceeds - cost
    opened = state.get("position_opened_at")
    trade = {
        "opened_at": opened,
        "closed_at": now_iso(),
        "entry_price": state.get("entry_price"),
        "exit_price": price,
        "btc": btc,
        "cost_gbp": cost,
        "proceeds_gbp": proceeds,
        "pnl_gbp": pnl,
        "pnl_pct": pnl / cost if cost else 0.0,
        "reason": reason,
    }
    state["cash_gbp"] += proceeds
    state["btc"] = 0.0
    state["entry_price"] = None
    state["position_cost"] = 0.0
    state["highest_price"] = None
    state["position_opened_at"] = None
    state["trades"].insert(0, trade)
    state["trades"] = state["trades"][:200]
    state["last_signal"] = "PAPER SELL"
    state["last_reason"] = reason
    record_event("SELL", f"Simulated SELL BTC at £{price:,.2f}; P/L £{pnl:.2f}. {reason}")
    save_state(state)
    snap = portfolio_snapshot(state, price)
    await send_email_async(
        f"PAPER SELL — P/L £{pnl:.2f}",
        f"Vincent's Bitcoin Paper Bot simulated a SELL.\n\n"
        f"Pair: {DISPLAY_PAIR}\nExit price: £{price:,.2f}\nProceeds after fee: £{proceeds:.2f}\n"
        f"Trade P/L: £{pnl:.2f} ({trade['pnl_pct']*100:.2f}%)\nReason: {reason}\n"
        f"Virtual equity: £{snap['equity_gbp']:.2f}\nTotal P/L: £{snap['pnl_gbp']:.2f}\n\nNo real money was used.",
    )


async def evaluate_market() -> None:
    candles, price = await fetch_market()
    closes = [c["close"] for c in candles]
    ema20 = ema(closes[-120:], 20)
    ema50 = ema(closes[-160:], 50)
    current_rsi = rsi(closes[-80:], 14)
    momentum3 = (closes[-1] / closes[-4] - 1) if closes[-4] else 0.0
    previous = candles[-2]
    current = candles[-1]

    with state_lock:
        state["last_price"] = price
        state["last_check_at"] = now_iso()
        state["last_error"] = None
        state["indicators"] = {
            "ema20": ema20,
            "ema50": ema50,
            "rsi14": current_rsi,
            "momentum3": momentum3,
            "interval_min": INTERVAL_MIN,
        }
        if state["btc"] > 0:
            state["highest_price"] = max(state.get("highest_price") or price, price)
        save_state(state)

    if not state["running"]:
        state["last_signal"] = "PAUSED"
        state["last_reason"] = "The bot is paused. Market data is still updating."
        save_state(state)
        return

    if state["btc"] <= 0:
        trend_up = ema20 > ema50
        price_above_fast = price > ema20
        healthy_rsi = 45 <= current_rsi <= 68
        rebound = current["close"] > previous["close"] and previous["low"] <= ema20 * 1.004
        positive_momentum = momentum3 > 0
        if trend_up and price_above_fast and healthy_rsi and rebound and positive_momentum:
            reason = (
                f"Uptrend confirmed (EMA20 £{ema20:,.0f} > EMA50 £{ema50:,.0f}), "
                f"RSI {current_rsi:.1f}, and price rebounded near the fast average."
            )
            await buy(price, reason)
        else:
            state["last_signal"] = "HOLD CASH"
            failed = []
            if not trend_up: failed.append("main trend is not positive")
            if not price_above_fast: failed.append("price is below EMA20")
            if not healthy_rsi: failed.append(f"RSI {current_rsi:.1f} is outside 45–68")
            if not rebound: failed.append("no confirmed pullback rebound")
            if not positive_momentum: failed.append("short momentum is not positive")
            state["last_reason"] = "; ".join(failed) or "Waiting for a stronger setup."
            save_state(state)
    else:
        entry = float(state["entry_price"])
        highest = float(state["highest_price"])
        change = price / entry - 1
        trail_change = price / highest - 1
        if change <= -STOP_LOSS_PCT:
            await sell(price, f"Stop-loss triggered at {change*100:.2f}% from entry.")
        elif change >= TAKE_PROFIT_PCT:
            await sell(price, f"Take-profit reached at {change*100:.2f}% from entry.")
        elif highest > entry * 1.008 and trail_change <= -TRAILING_STOP_PCT:
            await sell(price, f"Trailing stop triggered: {trail_change*100:.2f}% below the position high.")
        elif ema20 < ema50 and current_rsi < 45:
            await sell(price, f"Trend reversal: EMA20 below EMA50 and RSI {current_rsi:.1f}.")
        else:
            state["last_signal"] = "HOLD BTC"
            state["last_reason"] = (
                f"Open paper position is {change*100:.2f}% from entry; "
                f"highest observed price £{highest:,.2f}."
            )
            save_state(state)

    await maybe_daily_email()


async def maybe_daily_email() -> None:
    local_now = datetime.now(LOCAL_TZ)
    today = local_now.date().isoformat()
    if local_now.hour < DAILY_EMAIL_HOUR or state.get("last_daily_email_date") == today:
        return
    snap = portfolio_snapshot(state)
    body = (
        f"Daily paper-trading update for {DISPLAY_PAIR}.\n\n"
        f"Last price: £{(state.get('last_price') or 0):,.2f}\n"
        f"Signal: {state.get('last_signal')}\nReason: {state.get('last_reason')}\n"
        f"Virtual cash: £{state['cash_gbp']:.2f}\nVirtual BTC: {state['btc']:.8f}\n"
        f"Virtual equity: £{snap['equity_gbp']:.2f}\nTotal P/L: £{snap['pnl_gbp']:.2f} ({snap['pnl_pct']*100:.2f}%)\n"
        f"Completed trades: {len(state['trades'])}\n\nNo real money was used."
    )
    sent = await send_email_async(f"Bitcoin Paper Bot daily update — £{snap['equity_gbp']:.2f}", body)
    if sent:
        state["last_daily_email_date"] = today
        record_event("EMAIL", "Daily summary email sent.")
        save_state(state)


async def bot_loop() -> None:
    while True:
        try:
            await evaluate_market()
        except asyncio.CancelledError:
            raise
        except Exception as exc:
            with state_lock:
                state["last_error"] = str(exc)
                state["last_check_at"] = now_iso()
                record_event("ERROR", str(exc))
                save_state(state)
        await asyncio.sleep(CHECK_SECONDS)


@app.on_event("startup")
async def startup() -> None:
    global loop_task
    loop_task = asyncio.create_task(bot_loop())


@app.on_event("shutdown")
async def shutdown() -> None:
    if loop_task:
        loop_task.cancel()


@app.get("/health")
async def health() -> dict[str, Any]:
    return {"ok": True, "paper_trading": True, "last_check_at": state.get("last_check_at")}


@app.get("/api/status")
async def api_status(_: str = Depends(authenticate)) -> JSONResponse:
    with state_lock:
        payload = json.loads(json.dumps(state))
    payload.update(portfolio_snapshot(payload))
    payload["email_enabled"] = email_enabled()
    payload["pair"] = DISPLAY_PAIR
    payload["paper_trading"] = True
    payload["settings"] = {
        "starting_cash": STARTING_CASH,
        "position_pct": POSITION_PCT,
        "take_profit_pct": TAKE_PROFIT_PCT,
        "stop_loss_pct": STOP_LOSS_PCT,
        "trailing_stop_pct": TRAILING_STOP_PCT,
        "fee_rate": FEE_RATE,
        "check_seconds": CHECK_SECONDS,
    }
    return JSONResponse(payload)


@app.post("/api/pause")
async def pause(_: str = Depends(authenticate)) -> dict[str, Any]:
    state["running"] = False
    record_event("CONTROL", "Bot paused by user.")
    save_state(state)
    return {"ok": True, "running": False}


@app.post("/api/resume")
async def resume(_: str = Depends(authenticate)) -> dict[str, Any]:
    state["running"] = True
    record_event("CONTROL", "Bot resumed by user.")
    save_state(state)
    return {"ok": True, "running": True}


@app.post("/api/check-now")
async def check_now(_: str = Depends(authenticate)) -> dict[str, Any]:
    asyncio.create_task(evaluate_market())
    return {"ok": True}


@app.post("/api/test-email")
async def test_email(_: str = Depends(authenticate)) -> dict[str, Any]:
    if not email_enabled():
        raise HTTPException(400, "Email is not configured yet")
    sent = await send_email_async(
        "Bitcoin Paper Bot — test email",
        "The email connection works. This bot uses live Kraken market data and virtual money only.",
    )
    return {"ok": sent}


@app.post("/api/reset")
async def reset(_: str = Depends(authenticate)) -> dict[str, Any]:
    global state
    state = default_state()
    record_event("CONTROL", "Paper portfolio reset to starting balance.")
    save_state(state)
    return {"ok": True}


DASHBOARD_HTML = r'''<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Bitcoin Paper Bot</title>
<style>
:root{--bg:#07111f;--card:#101d2f;--line:#22344c;--text:#eef6ff;--muted:#91a4bc;--green:#38d996;--red:#ff6b79;--amber:#ffc857;--blue:#5aa9ff}
*{box-sizing:border-box} body{margin:0;background:radial-gradient(circle at 10% 0%,#15345b 0,#07111f 35%);color:var(--text);font:15px/1.45 system-ui,-apple-system,Segoe UI,sans-serif;min-height:100vh}
.wrap{max-width:1120px;margin:auto;padding:28px 18px 60px}.top{display:flex;gap:18px;justify-content:space-between;align-items:flex-start;flex-wrap:wrap}.brand h1{margin:0;font-size:clamp(25px,4vw,42px);letter-spacing:-1px}.brand p{margin:7px 0;color:var(--muted)}.badge{border:1px solid #2f9b73;background:#113b31;color:#7ff0c3;padding:9px 13px;border-radius:999px;font-weight:800}.grid{display:grid;grid-template-columns:repeat(4,1fr);gap:14px;margin-top:24px}.card{background:rgba(16,29,47,.94);border:1px solid var(--line);border-radius:17px;padding:18px;box-shadow:0 12px 35px #0004}.label{color:var(--muted);font-size:12px;text-transform:uppercase;letter-spacing:.1em}.value{font-size:27px;font-weight:850;margin-top:5px}.small{font-size:13px;color:var(--muted)}.wide{grid-column:span 2}.full{grid-column:1/-1}.signal{font-size:24px}.reason{margin-top:8px;color:#c9d6e5}.controls{display:flex;gap:10px;flex-wrap:wrap;margin-top:13px}button{border:0;border-radius:10px;padding:10px 14px;font-weight:800;cursor:pointer;background:var(--blue);color:#041426}button.danger{background:var(--red)}button.neutral{background:#293b53;color:white}button.good{background:var(--green)}table{width:100%;border-collapse:collapse;margin-top:12px;font-size:13px}th,td{text-align:left;padding:10px 8px;border-bottom:1px solid var(--line)}th{color:var(--muted)}.pos{color:var(--green)}.neg{color:var(--red)}.statusline{display:flex;gap:18px;flex-wrap:wrap;color:var(--muted);font-size:13px;margin-top:12px}.dot{display:inline-block;width:8px;height:8px;border-radius:50%;background:var(--green);margin-right:6px}.email-off{color:var(--amber)}.footer{margin-top:20px;color:var(--muted);font-size:12px}@media(max-width:800px){.grid{grid-template-columns:repeat(2,1fr)}.wide{grid-column:span 2}}@media(max-width:500px){.grid{grid-template-columns:1fr}.wide,.full{grid-column:span 1}.value{font-size:24px}}
</style>
</head><body><main class="wrap">
<div class="top"><div class="brand"><h1>Bitcoin Paper Bot</h1><p>Live Kraken market · Virtual money · Built for Vincent Salvatore</p></div><div class="badge">NO REAL MONEY</div></div>
<div class="grid">
<div class="card"><div class="label">BTC / GBP</div><div class="value" id="price">—</div><div class="small" id="check">Waiting…</div></div>
<div class="card"><div class="label">Virtual equity</div><div class="value" id="equity">—</div><div class="small">Starting balance £1,000</div></div>
<div class="card"><div class="label">Total P/L</div><div class="value" id="pnl">—</div><div class="small" id="pnlpct">—</div></div>
<div class="card"><div class="label">Position</div><div class="value" id="position">—</div><div class="small" id="cash">—</div></div>
<div class="card wide"><div class="label">Current decision</div><div class="value signal" id="signal">—</div><div class="reason" id="reason">—</div></div>
<div class="card wide"><div class="label">Indicators</div><div class="statusline"><span>EMA20 <b id="ema20">—</b></span><span>EMA50 <b id="ema50">—</b></span><span>RSI14 <b id="rsi">—</b></span></div><div class="controls"><button class="good" onclick="act('/api/resume')">Start</button><button class="danger" onclick="act('/api/pause')">Pause</button><button onclick="act('/api/check-now')">Check now</button><button class="neutral" onclick="act('/api/test-email')">Test email</button><button class="neutral" onclick="resetBot()">Reset simulation</button></div><div class="small" style="margin-top:10px" id="email">—</div></div>
<div class="card full"><div class="label">Completed paper trades</div><div style="overflow:auto"><table><thead><tr><th>Closed</th><th>Entry</th><th>Exit</th><th>P/L</th><th>Reason</th></tr></thead><tbody id="trades"><tr><td colspan="5" class="small">No completed trades yet.</td></tr></tbody></table></div></div>
<div class="card full"><div class="label">Recent activity</div><div id="events" class="small" style="margin-top:10px">No events yet.</div></div>
</div><div class="footer">Educational paper-trading experiment. It does not place orders and does not provide a guarantee of profit.</div>
</main><script>
const money=n=>new Intl.NumberFormat('en-GB',{style:'currency',currency:'GBP'}).format(n||0);const pct=n=>(100*(n||0)).toFixed(2)+'%';
async function load(){try{const r=await fetch('/api/status');if(!r.ok)throw Error('status '+r.status);const s=await r.json();
price.textContent=money(s.last_price);equity.textContent=money(s.equity_gbp);pnl.textContent=money(s.pnl_gbp);pnl.className='value '+(s.pnl_gbp>=0?'pos':'neg');pnlpct.textContent=pct(s.pnl_pct);
position.textContent=s.btc>0?s.btc.toFixed(7)+' BTC':'CASH';cash.textContent='Cash '+money(s.cash_gbp);signal.textContent=s.last_signal;reason.textContent=s.last_reason;check.textContent=s.last_check_at?'Updated '+new Date(s.last_check_at).toLocaleString():'Waiting for first check';
ema20.textContent=money(s.indicators.ema20);ema50.textContent=money(s.indicators.ema50);rsi.textContent=s.indicators.rsi14?s.indicators.rsi14.toFixed(1):'—';email.innerHTML=s.email_enabled?'<span class="dot"></span>Email alerts active':'<span class="email-off">Email alerts ready but not connected</span>';
trades.innerHTML=s.trades.length?s.trades.map(t=>`<tr><td>${new Date(t.closed_at).toLocaleString()}</td><td>${money(t.entry_price)}</td><td>${money(t.exit_price)}</td><td class="${t.pnl_gbp>=0?'pos':'neg'}">${money(t.pnl_gbp)} (${pct(t.pnl_pct)})</td><td>${t.reason}</td></tr>`).join(''):'<tr><td colspan="5" class="small">No completed trades yet.</td></tr>';
events.innerHTML=s.events.length?s.events.slice(0,10).map(e=>`<div style="padding:6px 0;border-bottom:1px solid #22344c"><b>${e.kind}</b> · ${new Date(e.time).toLocaleString()} · ${e.message}</div>`).join(''):'No events yet.';
}catch(e){reason.textContent='Dashboard error: '+e.message}}
async function act(url){const r=await fetch(url,{method:'POST'});let j={};try{j=await r.json()}catch{};if(!r.ok)alert(j.detail||'Action failed');setTimeout(load,700)}
function resetBot(){if(confirm('Reset the virtual portfolio and delete simulated trade history?'))act('/api/reset')}
load();setInterval(load,15000);
</script></body></html>'''


@app.get("/", response_class=HTMLResponse)
async def dashboard(_: str = Depends(authenticate)) -> HTMLResponse:
    return HTMLResponse(DASHBOARD_HTML)
