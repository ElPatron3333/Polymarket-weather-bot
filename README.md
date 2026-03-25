 # Polymarket Weather Bot                                                                                                                                                                                                                        A Python trading bot for Polymarket US temperature prediction markets. Identifies statistical discrepancies between     retail market prices and official NOAA NWS forecasts, then places limit orders when expected edge clears a              risk-adjusted threshold.

  > **Core signal logic is not public.** Architecture and design are documented here.
 ## How It Works

  **Pipeline: Fetch → Parse → Model → Compare → Execute**

  ```mermaid
  flowchart TD
      A[Polymarket CLOB API\nActive temperature markets] --> C[Market Parser\nMap markets to settlement stations]
      B[NOAA NWS API\napi.weather.gov] --> D[Forecast Engine\nBuild probability distribution]
      E[METAR / aviationweather.gov\nCurrent observations] --> D
      C --> F[Edge Calculator\nmodel_prob − market_price]
      D --> F
      F --> G[Trade Quality Scorer\nedge × prob × proximity × price × time]
      G --> H{Quality ≥ threshold?}
      H -- No --> I[Skip / Log]
      H -- Yes --> J[Risk Manager\nPosition caps, daily cap, spread filter]
      J --> K{Passes risk checks?}
      K -- No --> I
      K -- Yes --> L[Order Placer\nKelly-sized limit order via CLOB API]
      L --> M[Order Monitor\nCancel stale or forecast-shifted orders]
  ```
 ## Data Sources — Three-Oracle System

  No single data source is trusted unconditionally. The bot blends three independent oracles:

  | Source | Role |
  |--------|------|
  | **NOAA NWS** (`api.weather.gov`) | Primary forecast — official hourly temperature curve per city. Settlement source
  matches this, eliminating data mismatch risk. |
  | **ECMWF Ensemble** (via Open-Meteo) | 40–50 independent model runs per city. Extracts ensemble mean and spread
  (std). Wide spread = high disagreement = bot trades smaller or skips. |
  | **METAR** (`aviationweather.gov`) | Live airport station readings. Used same-day to adjust the forecast as the
  actual day unfolds — wind, cloud cover, current temperature trajectory. |

  The blended forecast feeds a Gaussian probability model: `P(bracket) = Φ((high − mean) / std) − Φ((low − mean) / std)`
   ## Signal Tiers — Tiered Entry System

  Not all edges are equal. Signals are classified into three tiers:

  | Tier | Condition | Position Size | Entry Style |
  |------|-----------|---------------|-------------|
  | **ALPHA** | Edge ≥ 10–15% (scales with lead time) | Full Kelly | Aggressive limit near ask |
  | **BETA** | Positive edge below ALPHA threshold | Half Kelly | Passive limit 3¢ below model prob |
  | **WINNER** | Market consensus bracket, slight negative edge acceptable | Small | Very passive, 4¢ below model prob |

  BETA signals require an ALPHA anchor on the same city-date — no orphan bets. WINNER signals capture value from being
  on the market consensus bracket even when model edge is thin.

  The bot also trades the **NO side**: when a bracket's YES price is ≥88¢, it evaluates whether the NO token offers edge
   using the same model and Kelly sizing.
    ## Protective Gates

  Before any order is placed, the signal must pass all of these checks:

  - **Ensemble spread gate** — if ECMWF ensemble std > 3.0°F, all models are disagreeing too much. Skip entirely.
  - **Cold front detection** — between 5AM–10AM local, if live METAR is ≥10°F below NWS's hourly expectation, a cold
  front has arrived that models haven't captured. Block all buys for that city-date for the rest of the day.
  - **Physical reachability** — given current observed temperature and time of day, can the temperature physically reach
   the bracket by peak time? If not, skip.
  - **ECMWF divergence gate** — if ECMWF and NWS means disagree by ≥3°F and the market agrees with ECMWF, downgrade
  ALPHA to BETA. At ≥5°F divergence, block entirely.
  - **Late-day gate** — after a certain hour, only trade the bracket immediately adjacent to the current market mode. No
   speculative positions far from the likely outcome.
  - **Stop-loss cooldown** — after a stop-loss exit, escalating cooldowns: ~16 min after 1st stop, ~60 min after 2nd,
  permanent block after 3rd. State persists across restarts via disk.
**Why NOAA NWS only?**
  Polymarket temperature markets settle against official airport ASOS/METAR readings. Using the same source as
  settlement (NWS) eliminates data source mismatch risk. Third-party weather APIs are excluded by design.

  **Why trade quality over raw edge?**
  Early versions ranked trades purely by `model_p − market_p`. This produced a bias toward cheap tail bets with high
  theoretical edge but poor win rates. The quality score penalizes low-probability brackets, contracts far from the
  blended forecast mean, ultra-cheap lottery tickets, and late-day uncertainty — resulting in higher-quality trade
  selection.

  **Why fractional Kelly with lead-time scaling?**
  Kelly criterion maximises long-run growth but is aggressive. Position sizes are scaled by a time-confidence multiplier
   that tightens as forecast horizon increases — full Kelly on same-day markets, down to 25% Kelly on day+6. This
  reflects the real decrease in forecast reliability over time.
   ## Position Sizing — Lead-Time Confidence Table

  ```python
  # Kelly multiplier and minimum edge threshold scale with forecast lead time.
  # Longer lead = more uncertainty = smaller bets, higher edge required.

  LEAD_CONFIDENCE = {
      0: {"edge_min": 0.12, "kelly_mult": 1.00},  # same-day:  full Kelly
      1: {"edge_min": 0.12, "kelly_mult": 0.90},  # tomorrow:  90% Kelly
      2: {"edge_min": 0.18, "kelly_mult": 0.70},  # day+2:     70% Kelly
      3: {"edge_min": 0.20, "kelly_mult": 0.60},  # day+3:     60% Kelly
      4: {"edge_min": 0.22, "kelly_mult": 0.40},  # day+4:     40% Kelly
      5: {"edge_min": 0.25, "kelly_mult": 0.30},  # day+5:     30% Kelly
      6: {"edge_min": 0.28, "kelly_mult": 0.25},  # day+6:     25% Kelly
 Markets beyond day+6 are skipped entirely — forecast reliability below threshold.
 ## Risk Controls

  | Control | Value | Purpose |
  |---------|-------|---------|
  | Per-market position cap | $100 | Limits exposure to any single bracket |
  | Per-city-date cap | $200 | Prevents over-concentration in one city |
  | Daily total cap | $1000 | Hard ceiling on total daily exposure |
  | Max bid-ask spread | 8¢ | Skips illiquid markets |
  | Min book depth | $50 | Ensures order fill is realistic |
  | Stale order TTL | 5 min | Cancels unfilled orders automatically |
  | Forecast shift detection | Live | Cancels open orders if NWS forecast moves against position |
  ## Tech Stack

  - **Python 3.11+**
  - `py-clob-client` — Polymarket CLOB API (order placement, position tracking)
  - `aiohttp` — async HTTP for parallel market + weather data fetching
  - `numpy` / `scipy` — probability distribution modeling (normal distribution over temperature brackets)
  - `eth-account` — wallet signing for on-chain order authentication
 

  ![Bot demo]![demo gif](https://github.com/user-attachments/assets/52c3ee4f-7562-414f-a4ba-8d74c83fe2c3)


 ## Setup

  ```bash
  pip install py-clob-client eth-account aiohttp numpy scipy python-dotenv tzdata
  cp .env.example .env   # fill in Polymarket API credentials
  python poly_weather_bot.py          # continuous loop
  python poly_weather_bot.py --once   # single scan and exit
  ```
## Available for Custom Work

  I build custom Polymarket bots and Python automation. Prediction markets, arbitrage detection, event-driven
  strategies, data pipelines.

  → DM on Discord / Telegram, or open an issue here.
  
