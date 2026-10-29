
# Crude Oil Price Volatility & Procurement Risk

When is it actually risky to buy oil? A real-data analysis of 40 years of crude oil price history.

## The problem

I'm approaching this as a data analyst for a company that regularly buys fuel or crude oil. My manager asks: oil prices swing up and down all the time, so when should we be worried, when is it safe to buy, and is this actually about supply running low, or is something else driving the price?

## Data source

Real, live data from the **U.S. Energy Information Administration (EIA)**, pulled directly from their [public API](https://www.eia.gov/opendata/) no downloaded file. 2,126 weeks of WTI prices (1986-2026), 2,055 weeks of Brent prices (1987-2026), and 1,279 months of U.S. crude oil production.

## Steps I took

1. Connected straight to EIA's live API and pulled weekly WTI and Brent spot prices.
2. Hit a real API problem along the way: EIA's system rejected my first attempt because it only accepts a specific set of parameter names (`frequency`, `data`, `facets`, `start`, `end`, `sort`, `offset`, `length`, `api_key`), not a plain `series_id` the way I first tried. Fixed it by moving the series code into the correct `facets` parameter.
3. Checked the data before trusting it no missing values, no duplicate dates across either series.
4. Calculated week-to-week percentage price changes for both WTI and Brent, plus the dollar gap between the two.
5. Also pulled U.S. crude oil production data, to test whether production levels actually explain price moves.
6. Loaded everything into SQL (SQLite) and asked real business questions.
7. Charted the results.

## Key code

```python
import requests
import pandas as pd

def get_eia_series(series_id, frequency="weekly"):
    url = "https://api.eia.gov/v2/petroleum/pri/spt/data/"
    params = {
        "api_key": API_KEY,
        "frequency": frequency,
        "data[0]": "value",
        "facets[series][]": series_id,
        "sort[0][column]": "period",
        "sort[0][direction]": "asc",
        "length": 5000,
    }
    resp = requests.get(url, params=params)
    rows = resp.json()["response"]["data"]
    df = pd.DataFrame(rows)
    df["period"] = pd.to_datetime(df["period"])
    df["value"] = pd.to_numeric(df["value"], errors="coerce")
    return df[["period", "value"]].rename(columns={"value": series_id})

wti = get_eia_series("RWTC")      # WTI crude oil spot price
brent = get_eia_series("RBRTE")   # Brent crude oil spot price
```

```sql
-- The 10 single weeks with the biggest price jumps
SELECT period, ROUND(RWTC, 2) AS wti_price, ROUND(wti_pct_change, 2) AS pct_change
FROM oil_prices
ORDER BY wti_pct_change DESC
LIMIT 10;
```

## What I found

| Metric | Value |
|---|---|
| Average weekly price change | 3.4% |
| Worst single-week spike | +373% (week of May 1, 2020) |
| Worst single-week drop | -83.5% (week of Apr 24, 2020) |
| WTI-Brent correlation | 0.99 |
| Average Brent premium over WTI | $1.74/barrel |
| Current Brent premium (Sept 2026) | $23.51/barrel |
| Production-to-price correlation | 0.27 |

- **Prices swing more than they look like on average.** The worst single-week spike was +373% and the worst drop was -83.5%, both in April-May 2020 — the real COVID oil crash, not a data error.
- **The data found real history on its own.** Without being told anything about world events, the "riskiest weeks" query surfaced the 2020 COVID crash, the 2008-2009 financial crisis, the 1990 Gulf War, 1998's Asian financial crisis, and 2022's Russia-Ukraine invasion.
- **WTI and Brent move almost identically** (0.99 correlation) tracking one gives a reliable picture of the other.
- **U.S. production barely explains price moves** (only 0.27 correlation) — global demand and geopolitics matter far more than domestic output.
- **A real, current anomaly:** as of late September 2026, Brent was trading about $23 above WTI, far above the $1.74 historical average verified against real news, tied to 2026 Strait of Hormuz tension.

## What I'd recommend

1. Don't assume stability even normal weeks can see double-digit swings.
2. WTI alone is a reasonable single indicator to track, given how closely it mirrors the global price.
3. Watch the Brent-WTI gap itself as an early warning sign when it widens sharply, it usually reflects real geopolitical or supply risk.

## Limits of this analysis

- This is weekly data, not daily short-term spikes within a week aren't visible.
- These are spot prices, not the negotiated contract prices most real buyers pay.
- Correlation is not causation, especially for the production comparison.
- This describes historical patterns only  it doesn't predict future prices.

## Tools used

Python (pandas), SQL (SQLite), EIA Open Data API, Google Colab, Looker Studio

## Links

- Live dashboard: *(add Looker Studio link once published)*
- Case study write-up: *(add Notion link)*
- Notebook: `oil_price_volatility_analysis.ipynb` (in this repo)

---
*Data source: U.S. Energy Information Administration (EIA), Petroleum Spot Prices, pulled live via the EIA API.*
```
