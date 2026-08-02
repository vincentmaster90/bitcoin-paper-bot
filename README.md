# Bitcoin Paper Bot

A no-real-money BTC/GBP paper-trading bot using Kraken's public OHLC endpoint.

- No Kraken API key
- No private endpoints
- No order placement code
- Live market checks every 5 minutes
- Trend/pullback entry and risk-managed exits
- Password-protected dashboard
- Optional SMTP email alerts

Run locally:

```bash
pip install -r requirements.txt
DASHBOARD_PASSWORD=change-this uvicorn app.main:app --host 0.0.0.0 --port 8000
```
