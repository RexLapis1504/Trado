# Trado: research report and build plan (draft for approval)

Team "Money Follows", NMIMS STME. Bloomberg Global Trading Challenge 2026 (GTC-2026), Asia & Oceania Group B.
Prepared 23 Oct 2026. Status: **plan only. No code until Rex approves.**
This is analysis for a simulated contest and a learning project. It is not financial advice.

How the facts were checked: GitHub repo metadata (stars, licence, last push, open issues) came from the GitHub API on 23 Oct 2026. Release versions and dates came from the PyPI JSON API on the same day. A few points came from README pages and web searches; each is linked. Star counts change daily, so treat them as approximate.

---

## 0. Summary

- **Don't adopt a big backtesting framework for the contest.** Our core jobs are event studies (the own-record test), bootstrap Monte Carlo and probability-of-touch maths. Plain **pandas/NumPy on a DuckDB + Parquet cache** is the fastest option to build and to run. Lean and NautilusTrader are strong engines, but each needs days of setup and data conversion. That is the wrong trade for 3 weeks.
- **Top picks:** yfinance + exchange_calendars for data, DuckDB/Parquet for storage, TA-Lib to cross-check our indicators, skfolio for clustering and risk, quantstats for reports, Streamlit for the dashboard. All are permissive licences (Apache, MIT or BSD).
- **Licence traps to avoid:** backtrader (GPL-3.0, last release 2023), backtesting.py (AGPL-3.0), NautilusTrader (LGPL-3.0, fine to use but note it), vectorbt (Apache-2.0 **plus Commons Clause**, so no selling), ArcticDB (Business Source Licence).
- **Dead or risky:** the original `twopirllc/pandas-ta` GitHub repo now returns 404. PyPI only has a 0.4.x beta (Sep 2025). Don't depend on it. FinRL's last PyPI release was Apr 2024.
- **Global coverage (§1.5):** a live test of Yahoo's free chart API returned daily and 1-minute data for **47 of 48 equity markets tested** across the Americas, Europe, Asia-Pacific and the Middle East & Africa. The Philippines is the one real gap. A single `exchanges.yaml` maps Bloomberg codes (e.g. `JT`, `BZ`, `LN`) to Yahoo suffixes, calendars, currencies and minor-unit divisors.
- **Two biggest modelling gaps to fix:** (1) model the "Next Tick" ~20-minute fill delay with 1-minute bars; (2) show every number with its sample size, a confidence interval, and a comparison against the base rate of the same universe. That base-rate comparison also reduces the survivorship problem.

---

## 1. Research report

### 1.1 Comparison table

Fit score (1–5) = how well it fits **our** needs: a contest, free data, Python, non-developers reading the output.

| Project | Category | Licence | Stars | Last push / last release | Maintained? | Fit | Why |
|---|---|---|---|---|---|---|---|
| [QuantConnect Lean](https://github.com/QuantConnect/Lean) | Backtest engine (event-driven, C# core, Python API) | Apache-2.0 | 22.2k | push 23 Oct 2026 / `lean` CLI 1.0.230, 15 Oct 2026 | Yes, very | 2 | Handles multiple currencies, exchange calendars and realistic fills. But data must be converted to Lean's format and the C# toolchain is heavy. Too slow to set up for the contest. |
| [NautilusTrader](https://github.com/nautechsystems/nautilus_trader) | Backtest engine (event-driven, Rust core) | **LGPL-3.0** | 30.7k | push 23 Oct 2026 / 1.231.0, 2 Aug 2026 | Yes, very | 2 | Supports [multi-currency cash accounts](https://nautilustrader.io/docs/latest/concepts/overview/) and stop orders, with nanosecond event timing. Needs Python 3.12+ and has a steep learning curve. Worth a look after the contest. |
| [backtrader](https://github.com/mementum/backtrader) | Backtest engine (event-driven) | **GPL-3.0** | 23.6k | push 19 Aug 2024 / 1.9.78.123, Apr 2023 | **No** (issues disabled, no releases since 2023) | 1 | Abandoned, with a GPL licence. Drop it. |
| [backtesting.py](https://github.com/kernc/backtesting.py) | Backtest engine (single asset) | **AGPL-3.0** | 9.1k | push 5 Aug 2026 / 0.6.6, 22 Jul 2026 | Yes, slow pace | 1 | Tests one ticker at a time and has an AGPL licence. Doesn't fit a 650-name, multi-currency setup. |
| [vectorbt](https://github.com/polakowo/vectorbt) (open-source edition) | Vectorised backtesting and signal research | Apache-2.0 + **[Commons Clause](https://github.com/polakowo/vectorbt/blob/master/LICENSE.md)** | 9.5k | push 23 Oct 2026 / 1.2.0, 19 Oct 2026 | Yes | 3 | Very fast parameter sweeps across many tickers. Commons Clause bans selling it, which is fine for us. Optional in phase 2 for sweeps; we don't need it in phase 1. |
| [zipline-reloaded](https://github.com/stefan-jansen/zipline-reloaded) | Backtest engine (event-driven, daily) | Apache-2.0 | ~2.0k | 3.2.0, 18 Oct 2026 | Bug fixes only; new work goes to ml4t-backtest | 2 | Built around US data bundles; its quickstart uses a Quandl key. Converting currencies across 7 markets would be painful. |
| [bt](https://github.com/pmorissette/bt) (+ [ffn](https://github.com/pmorissette/ffn)) | Portfolio-level backtesting | MIT | 3.0k (ffn 2.7k) | push 20 Oct 2026 / 1.4.1, 10 Oct 2026 | Yes (14 open issues) | 3 | Simple tree of weights and rebalancing, MIT licence. **Fallback engine** for whole-portfolio "what if" tests after the contest. |
| [ml4t-backtest](https://pypi.org/project/ml4t-backtest/) | Event-driven backtesting | check before use | new | 0.1.0b21 (beta) per [PyPI stats](https://pypistats.org/packages/ml4t-backtest) | Early beta | 1 | Stefan Jansen's successor to zipline. Too young to trust for now. |
| [Microsoft Qlib](https://github.com/microsoft/qlib) | ML factor research platform | MIT | 49.6k | push 22 Oct 2026 / 0.9.7, Aug 2025 | Repo active; PyPI releases are infrequent | 1 | Its data pipeline centres on China A-shares, and it needs a lot of setup. Interesting to learn from after the contest; no use during it. |
| [FinRL](https://github.com/AI4Finance-Foundation/FinRL) | Reinforcement-learning trading | MIT | 16.8k | push 23 Oct 2026 / 0.3.7, **Apr 2024** | Repo active, releases stale | 1 | RL with ~3 weeks of live decisions and small data will overfit. Skip it. |
| [OpenBB (Open Data Platform)](https://github.com/OpenBB-finance/OpenBB) | Data platform / research terminal | Apache-2.0 | 74.6k | 5.0.0, 29 Sep 2026 | Yes | 2 | A data-access layer with many providers, but most providers need API keys. The Workspace UI is a hosted product and isn't part of this repo. Note: GitHub currently serves this repo under the `openbq-org/OpenBB` path, while PyPI still links `OpenBB-finance/OpenBB` (same repo ID). Check the org before installing anything from it. |
| [yfinance](https://github.com/ranaroussi/yfinance) | Market data loader | Apache-2.0 | 25.6k | push 22 Oct 2026 / 1.7.0, 26 Aug 2026 | Yes | **5** | Daily and 1-minute OHLC for all 7 markets, with FX via `XXXUSD=X` tickers. The README says Yahoo data is "[intended for personal use only](https://github.com/ranaroussi/yfinance)". OK for a student contest; don't redistribute the data. |
| [exchange_calendars](https://github.com/gerrymanoim/exchange_calendars) | Exchange calendars and sessions | Apache-2.0 | 681 | 4.13.2, 10 Mar 2026 | Yes | **5** | Covers XNYS, XLON, XAMS, XPAR, XCSE, XOSL, XTKS, XBOM, XKRX and BVMF, with time zones. Doesn't cover **NSE** (XNSE). |
| [pandas_market_calendars](https://github.com/rsheftel/pandas_market_calendars) | Exchange calendars | MIT | 1.0k | push 5 Oct 2026 / 5.5.0, 5 Oct 2026 | Yes | 4 | Wraps exchange_calendars and **adds an [NSE calendar](https://pandas-market-calendars.readthedocs.io/en/latest/calendars.html)**. Use it for India. |
| [TA-Lib (Python)](https://github.com/TA-Lib/ta-lib-python) | Indicators | BSD-2 | 12.3k | push 23 Oct 2026 / 0.8.2, 12 Oct 2026 | Yes | 4 | The industry-standard indicator library. Install it from wheels (check on Windows). We use it to cross-check our own SMA, RSI and volatility code. |
| [pandas-ta](https://www.pandas-ta.dev) | Indicators | unclear | — | GitHub repo **404**; PyPI 0.4.71b0 (beta), Sep 2025, Python ≥3.12 | **Uncertain** | 1 | The source repo is gone and only betas are on PyPI. Avoid it. A community fork, [pandas-ta-classic](https://github.com/xgboosted/pandas-ta-classic) (MIT, ~460 stars, 0.8.32 Sep 2026), exists if needed. |
| [ta (bukosabino)](https://github.com/bukosabino/ta) | Indicators | MIT | 5.2k | push 18 Mar 2026 / 0.11.0, **Nov 2023** | Slow | 2 | Pure pandas and easy to install, but releases are stale. |
| [quantstats](https://github.com/ranaroussi/quantstats) | Tear sheets, benchmark-relative stats | Apache-2.0 | 7.7k | push 27 Sep 2026 / 0.0.86, 27 Sep 2026 | Yes | **5** | One call builds an HTML report against a benchmark, which suits non-developers. |
| [pyfolio-reloaded](https://github.com/stefan-jansen/pyfolio-reloaded) | Tear sheets | Apache-2.0 | 616 | 0.10.0, 17 Oct 2026 | Bug fixes only | 3 | Fallback for quantstats. |
| [empyrical-reloaded](https://pypi.org/project/empyrical-reloaded/) | Risk metrics | Apache-2.0 | — | 0.7.0, 17 Oct 2026 | Bug fixes only | 3 | Clean metric functions (alpha, beta, drawdown, capture ratios). |
| [alphalens-reloaded](https://github.com/stefan-jansen/alphalens-reloaded) | Factor / signal evaluation | Apache-2.0 | 675 | 0.5.0, 17 Oct 2026 | Bug fixes only | 3 | Measures signal quality by forward-return quantiles. Phase 2: test whether "uptrend + 1M ≥ 8%" really ranks well across all names. |
| [skfolio](https://github.com/skfolio/skfolio) | Portfolio optimisation, risk, clustering | BSD-3 | 2.5k | push 23 Oct 2026 / 1.10.1, 23 Oct 2026 | Yes, very | **5** | Uses the scikit-learn style API. Has hierarchical clustering, risk measures and cross-validation. Good for theme concentration. |
| [Riskfolio-Lib](https://github.com/dcajasn/Riskfolio-Lib) | Portfolio optimisation and risk | BSD-3 | 4.6k | push 22 Oct 2026 / 7.4.0, 4 Oct 2026 | Yes | 4 | Has the widest range of risk measures and good cluster charts. Fallback for skfolio. |
| [PyPortfolioOpt](https://github.com/PyPortfolio/PyPortfolioOpt) | Portfolio optimisation | MIT | 6.1k | 1.6.0, 26 Feb 2026 (repo moved to the `PyPortfolio` org) | Yes, slower pace | 3 | Simple to use, but we don't need mean-variance optimisation in a 20%-cap contest. |
| [DuckDB](https://github.com/duckdb/duckdb) | Local analytical storage | MIT | 42.4k | push 23 Oct 2026 / 2.0.0, 21 Oct 2026 | Yes, very | **5** | A single file with no server. Queries Parquet directly. Works on Windows and Linux. |
| [ArcticDB](https://github.com/man-group/ArcticDB) | DataFrame time-series database | **Business Source Licence 1.1** | 2.6k | 6.28.1, 20 Oct 2026 | Yes | 2 | Fast, but the BSL licence limits some uses, and it's overkill for our ~650 names. |
| [Polars](https://pypi.org/project/polars/) | Fast DataFrames | MIT | — | 2.0.0, 6 Oct 2026 | Yes | 3 | Optional speed-up in phase 2. NumPy is already fast enough for our sizes. |
| [Streamlit](https://github.com/streamlit/streamlit) | Dashboard | Apache-2.0 | 46.1k | push 23 Oct 2026 / 1.66.1, 21 Oct 2026 | Yes, very | **5** | A page is just a Python script. Teammates open a URL with no setup. |
| [Dash](https://github.com/plotly/dash) | Dashboard | MIT | 24.5k | 4.4.1, 21 Jul 2026 | Yes | 3 | More control over layout, but more code than Streamlit. |
| [Panel](https://github.com/holoviz/panel) | Dashboard | BSD-3 | 5.8k | 1.10.0, 23 Oct 2026 | Yes (873 open issues) | 3 | Good with notebooks. Fallback for Streamlit. |
| [xbbg](https://pypi.org/project/xbbg/) | Bloomberg API wrapper | Apache-2.0 | — | — | Yes | ? | Only works **if** `blpapi` runs on a licensed Terminal PC. xbbg itself [grants no data rights](https://www.sourcepulse.org/projects/1297822). Older READMEs say API data [may not leave the local PC](https://pypi.org/project/xbbg/0.5.5/). See open question 1. |

Crypto-only bots (Freqtrade, Hummingbot, Jesse and similar) were skipped as the brief asked. None of their parts are needed: their equity-relevant parts (indicators, backtest loops) are covered above.

### 1.2 Top pick and fallback per category

| Category | Top pick | Why (one line) | Fallback |
|---|---|---|---|
| Backtesting engine | **Our own vectorised event-study code** (pandas/NumPy) | Our tests are signal-then-hold-15-sessions-with-stop, which is about 100 lines of array code and runs in seconds on 650 names. | bt (MIT) for portfolio tests in phase 2; NautilusTrader for full simulation later |
| Research / screening | Own screener + **alphalens-reloaded** (phase 2) | The screen is simple filters; alphalens adds an honest check of whether the signal works. | vectorbt for parameter sweeps |
| Indicators | Own pandas code, **cross-checked against TA-Lib** | SMA, RSI and volatility are 5 lines each; TA-Lib catches our mistakes. | pandas-ta-classic |
| Portfolio and risk | **skfolio** | Clustering, risk measures and a familiar API, all actively maintained. | Riskfolio-Lib |
| Performance reporting | **quantstats** | One-call HTML report against a benchmark. | pyfolio-reloaded + empyrical-reloaded |
| Data access | **yfinance** (chart endpoint) + **exchange_calendars** / **pandas_market_calendars** (NSE) | Free, covers all 7 markets and FX, with correct sessions. | Direct calls to the Yahoo chart API with a browser User-Agent (already works for us) |
| Local storage | **DuckDB + Parquet** | One file and no server; works on Windows and Linux. | Plain Parquet files |
| Dashboard | **Streamlit** | Fastest way to give non-developers a page. | Panel |
| ML / RL | **None during the contest** | n < 10 samples and 3 weeks left: ML would only fit noise. | Qlib / alphalens after 13 Nov as a learning project |

### 1.3 Answers to the research questions

**Q1. Which engine runs the own-record test and the leaderboard Monte Carlo fastest on ~650 multi-currency names?**
None of the frameworks; plain NumPy does. The own-record test is an event study: find the dates where the signal fires, take the next 15 sessions of USD returns minus ACWI returns, and apply a 12% stop. With 650 names × ~2,500 days in one NumPy array, that runs in about a second. The Monte Carlo bootstraps rows of a (days × positions) matrix of excess returns, so 100k paths also take seconds. Frameworks add order-book realism we don't need for daily signals. We model the platform's quirks ourselves (Q4).

**Q2. FX, exchange calendars, time zones (IST user; US, EU and Asia sessions)?**
- FX: pull `JPYUSD=X`, `EURUSD=X`, `GBPUSD=X`, `DKKUSD=X`, `NOKUSD=X`, `INRUSD=X`, `KRWUSD=X` and `BRLUSD=X` daily closes from Yahoo. Convert each stock's local close at that market's close (see the caveat in the risks list).
- Calendars: `exchange_calendars` for XNYS, XLON, XAMS, XPAR, XCSE, XOSL, XTKS, XBOM, XKRX and BVMF; `pandas_market_calendars` "NSE" for India.
- Time zones: store every timestamp in UTC and show it in IST (Asia/Kolkata) in the UI. The "no buys in first 60 min of US open" rule is then a check against the XNYS open time converted to IST, which moves when US clocks change on 1 Nov 2026.

**Q3. Reducing survivorship bias with free data?**
Free data can't fix it fully, but four steps make it much smaller:
1. **Compare against a control group, not zero.** Report the signal's excess return minus the excess return of *random* entry dates in the same universe. Survivorship inflates both, so most of the bias cancels out in the difference.
2. **Use point-in-time membership where it exists.** For example, the free [historical S&P 500 membership CSV](https://github.com/hanshof/sp500_constituents) (1996 onward), or rebuilding membership from the change log as [Robot Wealth describes](https://robotwealth.com/how-to-get-historical-spx-constituents-data-for-free/). There is no free source for delisted prices. Paid options are Norgate or CRSP ([background](https://concretumgroup.com/historical-constituents-of-an-equity-index-in-python-norgate-data/)).
3. **Freeze our universe from now on.** Save today's ~650 names and any future delistings to the local store, so the 2027 version has less bias.
4. **Label every result** "survivor universe: likely optimistic", and shrink small-sample means toward the control mean (see the odds-engine design).

**Q4. Modelling our platform's mechanics?**

| Mechanic | Model |
|---|---|
| "Next Tick" fill ~20 min after sending | Fill price = the 1-minute bar close at send time + 20 min (configurable, plus or minus jitter). Back-testing that against our real fills calibrates the delay. Report the expected slippage versus the send-time price. This would have flagged the NTRA fill at the day's high. |
| Stop = "Last Price < level" | Trigger on the first 1-minute **close** below the level (daily low as the fallback when no minute data exists), then fill at trigger time + delay. Gaps through the stop fill at the open + delay. |
| No partial closes | A position is open or fully closed. Resizing means close everything and re-enter, which costs two fills with delay. |
| 20% single-position cap | Check at ticket time: amount ≤ 20% × current portfolio value. Also flag positions that drift above 20% (open question 3). |
| Rel P&L vs WLS | Time-weighted portfolio return minus index return. Yahoo doesn't carry WLS, so the default proxy is **ACWI**. We calibrate the gap by importing WLS levels exported by hand from the Terminal (GP → export), when available. |
| Cash earns 0 | Cash leg return = 0, so cash is a drag when the index rises. |

**Q5. Ingesting Bloomberg exports without BLPAPI?**
- `.xlsx` (Show History, any screen exported to Excel): `pandas.read_excel` with **openpyxl**, plus a column-mapping file per screen (one YAML file per export type). Validate on import: required columns present, totals reconcile.
- Screens with no export (ANR, EVT, CN): a **typed CSV template** that teammates fill in. That's quicker and more reliable than OCR on screenshots. OCR (e.g. Tesseract) is a phase-2 nice-to-have only.
- If BLPAPI turns out to be allowed on the lab PC, **xbbg** can pull BDH/BDP data. Licence terms likely keep that data on the lab PC, so only derived results would leave it.

**Q6. What is realistic in 1 week versus after the contest?** See section 3.

### 1.4 Risks

| Risk | Where | What to do |
|---|---|---|
| Abandoned project | backtrader (no release since 2023); pandas-ta (repo 404); ta (no release since 2023); FinRL (no release since 2024) | Don't depend on them. |
| Licence trap | GPL (backtrader), AGPL (backtesting.py), LGPL (Nautilus), Commons Clause (vectorbt), BSL (ArcticDB) | Our core stays MIT, BSD or Apache. Fine while we only use it ourselves; check again if we ever publish. |
| Data terms | Yahoo is "[personal use only](https://github.com/ranaroussi/yfinance)"; Bloomberg data has its own terms | Keep the cache private; don't publish raw data. Commit derived results only. |
| Data-cost trap | OpenBB providers, Finnhub international data, Norgate/CRSP are paid | Phase 1 uses only Yahoo and Bloomberg screens. Check [Finnhub](https://dataglobehub.com/api-finder/finnhub-api/)/FMP free tiers before relying on them for earnings dates. Default for events: manual EVT import. |
| Yahoo breakage / rate limits | yfinance depends on unofficial endpoints | Cache everything locally; refresh only what's new; keep the direct chart-API client as the fallback. |
| Proxy mismatch | ACWI ≠ WLS (WLS includes small caps and is a price-return index) | Show "vs ACWI (proxy)" on every number until WLS levels are imported. |
| FX timing | Asian closes come hours before the US close | Convert at each market's close using the same-day FX close; note it as a small daily error. |
| Small samples | n < 10 own-record cases | Show the n, a confidence interval and a colour warning on every figure, plus a shrunk estimate (see the odds-engine design). |

### 1.5 Global market coverage (US, Brazil, Europe, Asia and the rest)

Goal: cover any stock that could be in the WLS index, the way Bloomberg does, using free data first.

**What was tested.** On 23 Oct 2026, from a Linux cloud box, I called Yahoo's chart API (`/v8/finance/chart`, browser User-Agent) for one large stock per exchange. I asked for daily data and 1-minute data over the last 5 days. The [Yahoo exchange list](https://help.yahoo.com/kb/SLN2310.html) gave the suffix, quote delay and data vendor (ICE) for each market. Calendars were checked by installing `exchange_calendars` 4.13.2, which ships 71 calendars, and `pandas_market_calendars` 5.5.0.

**Result: 47 of 48 equity markets tested returned daily and 1-minute data.** The Philippines was the only market that failed. A few single tickers also failed because I had the symbol wrong, not because the market is missing: Kakao moved to KOSPI, OPAP.AT now trades as ALWN.AT, and Maybank is `1155.KL`.

| Region | Market (Yahoo suffix) | Live test | Ccy (quote unit) | Calendar | Yahoo delay |
|---|---|---|---|---|---|
| **Americas** | US NYSE/Nasdaq (none) | ✅ AAPL | USD | XNYS | real-time |
| | Brazil B3 (.SA) | ✅ PETR4 | BRL | BVMF | 15 min |
| | Mexico (.MX) | ✅ WALMEX | MXN | XMEX | 20 min |
| | Canada TSX (.TO) / TSXV (.V) | ✅ RY | CAD | XTSE | real-time |
| | Chile (.SN) | ✅ FALABELLA | CLP | XSGO | 15 min |
| | Argentina (.BA) | ✅ YPFD | ARS | XBUE | 30 min |
| | Colombia (.CL) | ✅ ECOPETROL | COP | XBOG | 20 min |
| | Peru | ✅ via US listing (BAP, NYSE); Lima itself not on Yahoo's list | PEN / USD | XLIM | — |
| **Europe** | UK LSE (.L) | ✅ SHEL | **GBp (pence ÷100)** | XLON | 20 min |
| | Euronext Amsterdam (.AS), Paris (.PA), Brussels (.BR), Lisbon (.LS), Dublin (.IR) | ✅ ASML, MC, ABI, EDP, RYA | EUR | XAMS, XPAR, XBRU, XLIS, XDUB | 15 min |
| | Germany Xetra (.DE) | ✅ SAP | EUR | XETR | 15 min |
| | Switzerland SIX (.SW) | ✅ NESN | CHF | XSWX | 30 min |
| | Spain (.MC), Italy (.MI), Austria (.VI), Greece (.AT) | ✅ ITX, ENEL, EBS, ETE | EUR | XMAD, XMIL, XWBO, ASEX | 15–20 min |
| | Nordics: Copenhagen (.CO), Oslo (.OL), Stockholm (.ST), Helsinki (.HE) | ✅ NOVO-B, EQNR, VOLV-B, NOKIA | DKK, NOK, SEK, EUR | XCSE, XOSL, XSTO, XHEL | real-time / 15 min |
| | Poland (.WA), Turkey (.IS) | ✅ PKO, THYAO | PLN, TRY | XWAR, XIST | 15 min |
| **Asia-Pacific** | Japan Tokyo (.T) | ✅ 7203 | JPY | XTKS | 20 min |
| | India NSE (.NS) / BSE (.BO) | ✅ RELIANCE | INR | XNSE (pmc) / XBOM | NSE real-time |
| | Korea KOSPI (.KS) / KOSDAQ (.KQ) | ✅ 005930 (KOSPI), 247540 (KOSDAQ) | KRW | XKRX | 20 min |
| | Hong Kong (.HK) | ✅ 0700 | HKD | XHKG | 15 min |
| | China Shanghai (.SS) / Shenzhen (.SZ) | ✅ 600519, 000858 | CNY | XSHG (use for both) | 30 min |
| | Taiwan (.TW / .TWO) | ✅ 2330 | TWD | XTAI | 20 min |
| | Australia (.AX), New Zealand (.NZ) | ✅ BHP, FPH | AUD, NZD | XASX, XNZE | 20 min |
| | Singapore (.SI), Malaysia (.KL), Thailand (.BK), Indonesia (.JK) | ✅ D05, 1155, PTT, BBCA | SGD, MYR, THB, IDR | XSES, XKLS, XBKK, XIDX | 10–20 min |
| | Philippines (.PS) | ❌ equities not served (Yahoo lists only PSE *indices*) | PHP | XPHS | — |
| | Vietnam (.VN) | ✅ VCB | VND | **none**, derive from bars | 15 min |
| **Middle East & Africa** | Saudi Tadawul (.SR) | ✅ 2222 | SAR | XSAU | 15 min |
| | Israel (.TA) | ✅ TEVA | **ILA (agorot ÷100)** | XTAE | 20 min |
| | South Africa JSE (.JO) | ✅ NPN | **ZAc (cents ÷100)** | XJSE | 15 min |
| | Kuwait (.KW), Qatar (.QA), UAE DFM (.AE) | ✅ NBK, QNBK, EMAAR | **KWF (fils ÷1000)**, QAR, AED | **none**, derive from bars | 15 min |

Other things the tests showed:

| Item | Finding | Design consequence |
|---|---|---|
| Daily history | ~26 years (PETR4.SA and 7203.T both back to Jan 2000) when `period1`/`period2` are given. `range=max` quietly switches to monthly bars. | Always request daily data with explicit dates. |
| Intraday history | 1-min: only the **last ~30 days**, at most ~8 days per request. 5-min: 60 days. 1-hour: ~730 days. | **Archive 1-minute bars every night from day 1.** Otherwise the Next-Tick fill calibration loses data after 30 days. |
| FX | All 25 pairs tested (`XXXUSD=X`) returned data, including BRL, INR, KRW, TWD, ZAR, SAR, TRY, IDR and CLP. | One FX table covers every market. |
| Minor-unit quotes | GBp, ZAc, ILA and KWF are quoted in pence, cents, agorot and fils. | `exchanges.yaml` holds a `quote_divisor` per market. Getting this wrong makes a stock look 100× its real size and breaks the 20% cap. |
| Benchmarks | ACWI, ^GSPC, ^BVSP, ^N225, ^NSEI, ^KS11, ^STOXX, ^FTSE and ^HSI all work. | Default proxy is ACWI; local indices are available for "vs home market" views. |
| Calendars | `exchange_calendars` covers 69 equity/derivatives venues; `pandas_market_calendars` adds NSE. No calendar for Vietnam, Kuwait, Qatar, UAE or Egypt. | For those five, session dates come from the days that actually have bars. |
| Data quirks | Yahoo returns "symbol may be delisted" both for real delistings and for wrong suffixes. Tickers change too (e.g. OPAP.AT now trades as ALWN.AT). | Validate the universe file nightly and flag any symbol that stops returning data. |

**Mapping Bloomberg tickers to Yahoo.** We type Bloomberg tickers on the Terminal (e.g. `7203 JT Equity`), but Yahoo needs `7203.T`. One `config/exchanges.yaml` row per market holds the whole mapping:

```yaml
- bbg: JT        # Bloomberg exchange code (JP = Japan composite also maps here)
  yahoo: .T
  mic: XTKS      # calendar
  ccy: JPY
  quote_divisor: 1
  tz: Asia/Tokyo
- bbg: LN
  yahoo: .L
  mic: XLON
  ccy: GBP
  quote_divisor: 100   # Yahoo quotes in pence
```

I'll pre-fill the main codes: US, BZ, MM, CN/CT, CI, AR, LN, NA, FP, BB, PL, ID, GY, SW/SE, SM, IM, AV, GA, DC, NO, SS, FH, PW, TI, JT, IN/IS/IB, KS/KQ, HK, CH/C1/C2, TT, AU/AT, NZ, SP, MK, TB, IJ, VN, AB, IT, SJ, KK, QD and UH. These come from memory, so you need to **check them against real Terminal tickers before we rely on them** (one look per market). Fixing one is a one-line change. Some exceptions need extra care: share classes (`NOVOB DC` ↔ `NOVO-B.CO`), Hong Kong zero-padding (`700 HK` ↔ `0700.HK`), and Brazilian numeric suffixes (`PETR4 BZ` ↔ `PETR4.SA`).

**Building the universe (which names, not just which markets).** Bloomberg screens the full WLS index. With free data, three options:

| Option | Coverage | Cost | Note |
|---|---|---|---|
| A. Our current ~650-name list + additions from the Terminal (EQS screen → Excel export) | Whatever we screen | Free | **Default.** Run EQS with "member of WLS" and export it. That gives the true contest universe with Bloomberg tickers, which `exchanges.yaml` converts. |
| B. iShares ETF holdings CSVs (e.g. ACWI ≈ 2,300 names, plus country ETFs) | Large and mid caps in ~47 markets | Free download | Good for checking coverage. Its holdings use MSCI's list, which is close to WLS but not the same. Check the iShares site terms before automating it. |
| C. Paid all-world provider symbol lists | Full exchange listings | ~$20–30/month and up | Only if A and B fall short. |

**Free versus paid: where Bloomberg-like coverage really comes from.**

| Source | Markets | Intraday | Cost | Fit |
|---|---|---|---|---|
| **Yahoo chart API** (via yfinance or direct) | ~50 equity markets, tested above | 1-min for 30 days | Free; [personal use](https://github.com/ranaroussi/yfinance) | **Phase 1 primary** |
| **Bloomberg BLPAPI** (via xbbg) | Everything in WLS, with real WLS levels, events and ratings | Yes | Already paid by NMIMS *if* API use is allowed | **Best possible**; depends on open question 1 |
| LSEG Workspace / Eikon or FactSet (if NMIMS has a seat) | Global | Yes | University licence | Ask the library; same data-terms caveat as Bloomberg |
| [EODHD](https://eodhd.com/list-of-stock-markets) | 63 exchange entries listed (marketing says 70+); covers the gaps above, e.g. Philippines (PSE), Pakistan, Sri Lanka, several African markets | Plan-dependent | From ~$19.99/month; free tier very limited | **Paid fallback** if Yahoo breaks |
| [Twelve Data](https://datarade.ai/data-providers/twelve-data/profile) | Claims 250+ venues | Yes | International data from ~$29/month ([third-party figure](https://geekflare.com/guides/best-stock-market-api/); check the official page) | Alternative fallback |
| Finnhub / FMP free tiers | Mostly US on free plans | Limited | Free to paid | Earnings dates (US) only |

**Bottom line.** Yahoo already covers every market in our current list and almost every WLS country, for free and with 1-minute bars. The real gaps are:
- The Philippines.
- Calendars for Vietnam and the Gulf (worked around by deriving sessions from bars).
- Intraday history older than 30 days (fixed by archiving from now on).
- **WLS membership and index levels themselves.** Only Bloomberg has these, so they come from Terminal exports (EQS for members, GP for levels) unless BLPAPI is allowed.

---

## 2. Architecture

### 2.1 Module diagram and data flow

```mermaid
flowchart LR
  subgraph Sources
    Y[Yahoo chart API<br/>daily + 1-min OHLC, FX]
    BX[Bloomberg exports by hand<br/>Show History .xlsx, ANR, EVT, WLS levels]
    T[CSV templates<br/>events, analyst snapshot, board points]
  end

  subgraph Data layer
    ING[ingest/<br/>yahoo.py, bloomberg_xlsx.py, templates.py]
    CAL[calendars.py<br/>sessions, tz, holidays]
    FX[fx.py<br/>local to USD]
    DB[(DuckDB + Parquet<br/>prices, fx, events, ledger, runs)]
  end

  subgraph Analytics
    SCR[Screener]
    OWN[Own-record backtester]
    PORT[Portfolio model<br/>book, Rel P&L, caps, correlation]
    ODDS[Odds engine<br/>ranges, P(P&L >= 0), stop-touch]
    LB[Leaderboard model]
    EXE[Execution helper<br/>ticket sheet + warnings]
  end

  UI[Streamlit dashboard<br/>book, stops, events, odds, alerts]
  REP[Reports<br/>quantstats HTML, CSV, Markdown]

  Y --> ING
  BX --> ING
  T --> ING
  ING --> DB
  CAL --> FX --> DB
  DB --> SCR & OWN & PORT & ODDS
  PORT --> ODDS --> LB
  OWN --> ODDS
  SCR --> EXE
  PORT --> EXE
  ODDS --> EXE
  SCR & OWN & PORT & ODDS & LB & EXE --> UI
  PORT & OWN --> REP
```

### 2.2 Folder layout

```
Trado/
  pyproject.toml            # pinned deps; Python 3.12 (works on Windows and Linux)
  config/
    contest.yaml            # capital, caps, end date, delay minutes, stop %, benchmark proxy
    exchanges.yaml          # one row per market: Bloomberg code, Yahoo suffix, MIC calendar, ccy, quote_divisor, tz
    universe.csv            # names: bbg_ticker, yahoo_symbol (derived), sector, theme; from our list + EQS export
    themes.yaml             # oil, drugs, AI platforms -> tickers
    bloomberg_maps/         # column maps for each Bloomberg export type
  data/                     # gitignored: trado.duckdb + parquet cache + raw exports
  trado/
    data/      yahoo.py  calendars.py  fx.py  store.py  bloomberg.py  templates.py
    signals/   indicators.py  screener.py
    backtest/  own_record.py  fills.py (Next Tick + stop model)  stats.py (CI, shrinkage)
    portfolio/ ledger.py  book.py  caps.py  correlation.py
    odds/      ranges.py  session_odds.py  montecarlo.py  leaderboard.py
    execution/ ticket.py  checks.py
    report/    tearsheet.py  formatting.py
  app/
    Home.py                 # Streamlit one-pager
    pages/                  # Screener, Own record, Odds, Ledger import
  cli.py                    # `trado refresh`, `trado screen`, `trado odds PLTR --stop 177 --by 2026-11-02`
  tests/
    golden/                 # frozen inputs + expected outputs (INSW, PLTR, ledger sample)
  docs/
```

### 2.3 Key interfaces

Every result object carries `n`, `source` and `as_of`, so the UI can't show a number without them.

```python
# data
get_prices(tickers, start, end, interval="1d", currency="USD") -> DataFrame  # MultiIndex (ts_utc, ticker)
get_fx(ccy, start, end) -> Series
sessions(mic, start, end) -> DatetimeIndex

# signals
screen(universe, as_of, rules: ScreenRules) -> DataFrame       # one row per name, pass/fail per rule, USD excess vs benchmark

# own record
own_record(ticker | universe, signal: Signal, hold=15, stop=0.12,
           benchmark="ACWI", fill: FillModel) -> OwnRecordResult
# OwnRecordResult: n, mean, median, win_rate, worst, ci95, control_mean, shrunk_mean, warning, cases (DataFrame)

# portfolio
load_ledger(xlsx_path) -> Ledger                                 # validated trades
book(ledger, prices, as_of) -> Book                              # positions, weights, Rel P&L by idea
check_caps(book, rules) -> list[Alert]                           # single 20%, sector 20%, theme
correlation(book, lookback=60) -> DataFrame

# odds
close_range(ticker, horizon_sessions, method="bootstrap") -> RangeResult
p_touch(ticker, level, by_date, model="bootstrap|gbm") -> ProbResult
portfolio_odds(book, horizon, jumps: EventJumps, paths=100_000) -> ProbResult  # P(P&L>=0), P(Rel>=X)
rank_from_rel(rel, board: BoardSnapshot) -> RankResult           # top-10/50/100 lines

# execution
ticket(ticker, amount_usd, book, now_ist) -> Ticket              # shares, stop, entry window, warnings
```

---

## 3. Build plan

### 3.1 Phase 1: contest week (≤ 5 working days, smallest useful version)

Goal: replace rebuilding everything by hand in chat with a cached data store, five commands and one dashboard page.

| Day | Task | Effort | Done when |
|---|---|---|---|
| 1 | Project skeleton, `contest.yaml`, `universe.csv` (from our current list) | 2 h | `pip install -e .` works on Windows and Linux |
| 1 | Yahoo loader (daily + 1-min, retry, browser User-Agent) + DuckDB/Parquet cache + incremental refresh | 4 h | Full 650-name daily refresh < 10 min; repeat run < 1 min |
| 1 | Calendars (exchange_calendars + NSE + bar-derived sessions for VN/Gulf) + FX to USD + `exchanges.yaml` (Bloomberg↔Yahoo map, minor-unit divisors) | 3 h | Every price has a USD value and a valid session date; GBp/ZAc/ILA/KWF tests pass |
| 1 | Nightly archive job for 1-minute bars (Yahoo keeps only ~30 days) + universe check (flag symbols that stop returning data) | 1 h | Runs on a schedule; a missing symbol shows up as an alert |
| 2 | Indicators (SMA 20/50/200, slope, RSI, 1M/3M return, 60-day volatility), TA-Lib cross-check test | 3 h | Matches TA-Lib to 1e-6 |
| 2 | Screener + CSV/Streamlit table | 3 h | Same pass list as our last manual screen |
| 2–3 | Own-record backtester + stop + Next-Tick fill model + stats (CI, control group, shrinkage, warning) | 6 h | INSW golden test passes |
| 3 | Ledger import from Show History .xlsx; book; Rel P&L by idea; cap/sector/theme checks; correlation | 5 h | Reconciles to the Terminal's P&L within $1k |
| 4 | Odds engine: close ranges, stop-touch probability, portfolio P(P&L ≥ 0) / P(Rel ≥ X) via joint bootstrap + results-day jumps | 6 h | PLTR golden test passes |
| 4 | Leaderboard model from a hand-entered board CSV | 2 h | Shows rank lines for top 10/50/100 |
| 5 | Execution helper (ticket sheet + warnings: first-hour, caps, results inside the window, fill-delay risk) | 3 h | Warns correctly on 5 scripted cases |
| 5 | Streamlit one-pager (book, distance to stops, events next 10 days, odds, alerts) | 4 h | Teammates open it and read it without help |
| 5 | Golden tests + README "how to run" for teammates | 2 h | `pytest` passes; a teammate runs it from the README |

Total ≈ 44 hours. That's tight for 5 days alongside trading. **If time runs short, cut in this order:** leaderboard model, then correlation chart, then 1-minute fill model (fall back to daily open + slippage).

Phase 1 is **not** going to include: OCR, BLPAPI, ML, vectorbt, point-in-time universes, multi-user hosting.

### 3.2 Phase 2: after 13 Nov (full version)

| Task | Effort |
|---|---|
| Post-mortem: fill-delay calibration from all real fills; how good were our odds (Brier score of our forecasts) | 1 day |
| Point-in-time universe (S&P 500 history + frozen contest universe + delisting log) | 2 days |
| alphalens study of our signals over the whole universe and several years; vectorbt parameter sweeps with out-of-sample split | 2 days |
| skfolio clustering for themes (data-driven, not hand-made lists); risk contribution by position | 1 day |
| quantstats / pyfolio tear sheets per contest and per idea | 0.5 day |
| bt or NautilusTrader full-portfolio simulation with platform mechanics | 3–5 days |
| BLPAPI/xbbg connector if the licence allows; OCR for screenshots if still needed | 2 days |
| Hosted Streamlit for the team; scheduled refresh | 1 day |
| Docs + handover so next year's team can reuse it | 1 day |

### 3.3 Test plan

| Test | Type | Pass rule |
|---|---|---|
| **INSW own record** | Golden | Uptrend + ≥ 8% in a month → 15-session excess vs ACWI, 12% stop: **mean ≈ +3.1%, n = 16**. n must match exactly; mean within ±0.3 pp. If it doesn't match, write down which rule differs (signal date, fill price or stop check) before changing anything. |
| **PLTR stop-touch** | Golden | P(touch 177 before 2 Nov) ≈ **25%**, within ±5 pp, using the same lookback and method as the original estimate (record both in the test). |
| Ledger reconciliation | Golden | Sample Show History export → positions and Rel P&L match the Terminal screen. |
| Indicators | Unit | Equal to TA-Lib on random series. |
| Calendars / time zones | Unit | US first-hour rule correct across the 1 Nov 2026 DST change; NSE and KRX holidays respected. |
| Fill model | Unit | Signal at 14:30 UTC → fill uses the 14:50 bar; a gap through the stop fills at the open + delay. |
| Caps | Unit | 20.1% position → alert; sector 21% → warning (bendable), not a block. |
| Monte Carlo | Property | With a fixed seed the results repeat exactly; with zero-edge inputs, P(Rel ≥ 0) ≈ 50%. |
| Small-sample flag | Unit | n < 10 → red "not reliable"; 10–29 → amber; ≥ 30 → none. |
| End-to-end | Smoke | `trado refresh && trado screen && streamlit run app/Home.py` on a Windows laptop and in Linux. |

How small samples will be shown (applies everywhere):
- n, mean, median, win rate with a [Wilson](https://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval#Wilson_score_interval) interval, worst case.
- A 95% bootstrap CI for the mean.
- A "shrunk mean" pulled toward the control group by n / (n + k), where k is set from the data (empirical Bayes).
- The odds engine uses the **shrunk** edge by default. The full backtest edge is an option and is labelled as such, matching our current "optional half backtest edge".

---

## 4. Open questions

At most three of these change the design. The rest just need a quick yes or no.

1. **BLPAPI:** Does the NMIMS Terminal licence allow Desktop API / `blpapi` on the lab PC, and may derived results leave that PC? *If yes:* in phase 2 the Bloomberg data layer switches from hand exports to xbbg. *If no:* hand exports and templates stay.
2. **Show History export:** Can you send one sample `.xlsx` (it can be redacted)? The ledger importer's column map depends on its exact layout.
3. **20% cap timing:** Does Bloomberg apply the 20% limit only when a trade is placed, or also to drift afterwards? This sets whether drift is a hard alert or a note.

Quick confirmations:
- Can you check the Bloomberg exchange codes in §1.5 against real tickers, and send one EQS export of WLS members? That makes the contest universe exact.
- Does NMIMS also license LSEG Workspace/Eikon or FactSet? Either would be a second global source.
- Can we get WLS index daily levels (Terminal GP → export) once a week? Without them, all "Rel" numbers use ACWI as a proxy.
- Teammates' skills: will anyone besides you run Python, or do the other four only read the dashboard? This sets how much effort goes into hosting versus a local run.
- Is Python 3.12 OK on your Windows laptop? Several picks need ≥ 3.10–3.12.

---

## Sources

GitHub repos are linked in the table. Release data: PyPI JSON API (`https://pypi.org/pypi/<package>/json`), read 23 Oct 2026. Other sources:
- yfinance personal-use note: https://github.com/ranaroussi/yfinance
- vectorbt licence (Commons Clause): https://github.com/polakowo/vectorbt/blob/master/LICENSE.md
- NautilusTrader concepts (multi-currency accounts): https://nautilustrader.io/docs/latest/concepts/overview/
- exchange_calendars coverage: https://github.com/gerrymanoim/exchange_calendars
- pandas_market_calendars NSE: https://pandas-market-calendars.readthedocs.io/en/latest/calendars.html
- zipline-reloaded maintenance note: https://github.com/stefan-jansen/zipline-reloaded
- pyfolio / alphalens maintenance notes: https://github.com/stefan-jansen/pyfolio-reloaded, https://github.com/stefan-jansen/alphalens-reloaded
- ml4t-backtest: https://pypistats.org/packages/ml4t-backtest
- Survivorship data: https://github.com/hanshof/sp500_constituents, https://robotwealth.com/how-to-get-historical-spx-constituents-data-for-free/, https://concretumgroup.com/historical-constituents-of-an-equity-index-in-python-norgate-data/
- xbbg / Bloomberg API terms: https://pypi.org/project/xbbg/0.5.5/, https://www.sourcepulse.org/projects/1297822
- Finnhub tiers: https://dataglobehub.com/api-finder/finnhub-api/
- GTC rules background (2024/2025; 2026 rules not public at time of search): https://www.fa.mgt.tum.de/fm/teaching/project-studies/list-of-open-project-studies/bloomberg-global-trading-challenge-2026/, https://www.bloomberg.com/company/press/bloombergs-global-trading-challenge-sets-participation-record/
