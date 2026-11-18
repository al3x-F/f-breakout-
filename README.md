sp500-breakout
A daily-timeframe (1D) scanner that finds and ranks the most significant FRESH and DEVELOPING
bullish and bearish breakouts across the S&P 500, built around a quantitative rigidity score for
support/resistance levels. It is an analytical and research tool: LONG/SHORT labels describe detected market
conditions, not trading instructions.
> **Status:** the code was written without access to a Rust compiler (the page's JavaScript, by contrast, was
> smoke-tested in Node against a mocked API). The algorithms were cross-checked
> against a Python port (see "Verification"), but you should run `cargo test` first and expect to fix a few
> compile errors or warnings.
Getting the Windows .exe and the simple interface
Double-click `sp500-breakout.exe` (no arguments) and it starts a small local web server and opens your
browser: pick a data source, press Run scan, click any row for the explanation, score breakdown and chart.
There is also a Backtest tab. Demo data works immediately with no account; for live data paste an FMP key
into the page (kept in memory only) or set `FMP_API_KEY` / put it in a `.env` file next to the exe.
The server listens on `127.0.0.1` only, and API calls need a per-session token, so no other web page can
trigger scans with your key. A console window stays open while it runs; close it to quit.
The `.exe` has to be compiled on a machine with a Rust toolchain. Three ways:
No installs - GitHub Actions. Push this folder to a GitHub repository, open Actions -> Build Windows exe
-> Run workflow, then download the `sp500-breakout-windows` artifact (`.github/workflows/build-windows.yml`).
On your own Windows PC. Install Rust from https://rustup.rs (accept the defaults, including the
Visual Studio C++ Build Tools it offers), then double-click `build.bat`. The result is `dist\sp500-breakout.exe`.
Cross-compile from Linux/macOS. `rustup target add x86_64-pc-windows-gnu`, install `mingw-w64`, then
`cargo build --release --target x86_64-pc-windows-gnu`.
The release build is a single file (`strip = true`, page embedded with `include_str!`). `config.toml` is
optional and read from the folder the program is started in.
Downloading two years of data with the Python scraper (from inside the exe)
`tools/scrape_companies.py` scrapes public web pages and writes CSV files. The program embeds the script, so the
single `.exe` can run it for you:
In the page: open the Get data tab, choose how many companies (default 50; the whole index is ~500) and
how many years (default 2), paste your Stooq key, and click Start download. A progress bar and log follow the
script live and there is a Cancel button. When it finishes click Use this data for scanning, which selects
Local CSV files as the source.
From a terminal: `sp500-breakout scrape --count 200 --years 2` (or `--tickers AAPL,MSFT`, or `--profiles`).
Directly: `python tools/scrape_companies.py prices --count 200` (or `profiles`). Run `python -m unittest` inside
`tools/` for its offline tests.
What you need to provide:
Python 3.8 or newer installed on the PC (python.org; tick "Add python.exe to PATH"). The app looks for
`python`, `py -3`, then `python3` and tells you if none works. No packages are needed (standard library only).
A free Stooq API key. Stooq now issues keys through a CAPTCHA form on its data-download page. Paste it into
the page, or set the `STOOQ_API_KEY` environment variable / `.env` entry. It is passed to the script through the
environment (never on the command line), is not logged, and is stripped from every message.
Optionally your email or project URL in `[scrape] contact` of `config.toml`; Wikimedia asks scripts to identify
themselves in the User-Agent.
What the `prices` command writes (the exact layout the Local CSV files source reads):
`data/csv/<TICKER>.csv` (`date,open,high,low,close,adj_close,volume`, `adj_close = close`), `data/csv/_GSPC.csv` (the
S&P 500 index for the trend filter; falls back to `SPY.csv`) and `data/constituents.csv` (`symbol,name,sector`) for the
companies actually downloaded. The company list and sectors come from Wikipedia (CC BY-SA); prices from Stooq.
Behaviour worth knowing: it checks `robots.txt`, waits between requests (`[scrape] delay_seconds`), skips files
downloaded in the last 12 hours (so an interrupted run resumes), and if Stooq's daily quota is reached it stops, keeps
everything downloaded and exits with code 4 - run it again later. Exit codes: 0 ok, 1 failure, 2 usage, 3 key
missing/rejected, 4 quota. Stooq data is free for non-commercial use and is not an official exchange feed; check its
terms and verify split adjustment before relying on it. I could not reach the live sites from where this was written,
so the scraper is tested against fixture pages (26 offline tests) rather than the real Stooq/Wikipedia responses.
Using real market data
In the page choose Live market data (Financial Modeling Prep) and paste your API key (or set `FMP_API_KEY`).
Click Test connection first. It makes about five requests and tells you, in plain words, whether the key,
the constituent list, index prices and stock prices all work on your plan. If something fails, the message
says which step and why (wrong key, endpoint not in your plan, request limit).
Click Run scan. A progress line shows the download (about 505 requests).
Look up one ticker analyses any symbol (it need not be in the S&P 500) and draws its chart.
Plan notes (from FMP's published pricing at the time of writing; check the current page): the free Basic plan allows
250 calls per day, so a full scan does not fit - use Limit to first N stocks (about 200) or a paid plan. Starter
allows 300 calls per minute, so a full scan takes a couple of minutes (the app caps itself at 250/minute by default,
`data.max_requests_per_minute`). Whether your plan includes index prices and the historical index-change list is not
something I could confirm: if the index symbol returns nothing the app automatically uses `SPY` for the market-trend
filter, and without change history backtests fall back to today's constituents.
Quick start
```bash
# 1. Offline demo - no API key, no network
cargo run --release -- --source synthetic scan
cargo run --release                                     # browser interface (same as double-clicking the exe)

# 2. Real data (Financial Modeling Prep)
cp .env.example .env            # then put your key in .env   (FMP_API_KEY=...)
cargo run --release -- scan

# Other commands
cargo run --release -- chart AAPL                       # SVG chart for one ticker
cargo run --release -- backtest --start 2025-01-02      # replay history, report performance
cargo run --release -- --source synthetic backtest      # backtest on the demo market
cargo run --release -- universe                         # list constituents
cargo test                                              # unit + integration tests (offline)
```
Global flags: `--config FILE`, `--source fmp|csv|synthetic`, `--limit N` (first N constituents).
Set `RUST_LOG=debug` for data warnings. Outputs go to `output/` (JSON, CSV, and one SVG chart per ranked
signal); `state/signals.json` is the signal registry.
Data sources
source	needs	notes
`fmp`	`FMP_API_KEY` env var	constituents, historical index changes, dividend-adjusted EOD bars; cached
`csv`	files only	`data/constituents.csv` + `data/csv/<SYMBOL>.csv` (see `data/csv_source.rs`)
`synthetic`	nothing	deterministic 60-stock market with planted breakouts; for demos and tests
New providers implement the three-method `DataProvider` trait (`src/data/provider.rs`).
Module map (and the spec sections each implements)
```
src/
  config.rs              every threshold/weight/lookback; defaults = spec values        (all)
  types.rs               Bar, Direction, Classification, ...
  data/
    provider.rs          DataProvider trait, point-in-time MembershipHistory            (2.3, 11)
    fmp.rs               Financial Modeling Prep provider (key from env var)
    csv_source.rs        local-CSV provider + CSV helpers
    cache.rs             on-disk cache wrapper
    synthetic.rs         deterministic demo/test market
    calendar.rs          US/Eastern session-complete logic                              (2.3)
    integrity.rs         adjustment, cleaning, completeness, anomaly checks             (2.3)
  indicators/            SMA, RSI(14, Wilder), ATR(14, Wilder), Bollinger(20, 2)        (6)
  volume.rs              RVOL (20d), Z-score, abnormality score                         (5)
  levels/
    pivots.rs            ATR-prominence swing highs/lows
    zones.rs             ATR-adaptive zones, recency/volume weighted                    (3)
    rigidity.rs          interactions + 0-100 rigidity score                            (4)
  market.rs              S&P 500 50/200-day trend filter                                (8.1)
  consolidation.rs       duration / range / touches / volume & volatility contraction   (8.2)
  breakout/
    context.rs           indicator bundle for a bar slice
    detector.rs          first buffered close beyond an eligible zone                   (7.1-7.3)
    classifier.rs        FRESH / DEVELOPING / EXTENDED / FADING / FAILED                (7.4-7.5)
    continuation.rs      post-breakout base + resumption                                (7.5)
    mod.rs               Analyzer: the whole per-stock pipeline
  scoring.rs             normalised components + explicit point adjustments             (9)
  signal.rs              evaluated signal + plain-English explanation                   (10)
  registry.rs            cross-run memory, reclassification tracking                    (12)
  scanner.rs             universe orchestration; the FRESH/DEVELOPING gate              (10)
  report.rs / chart.rs   table, JSON, CSV; SVG candlestick/volume/RSI charts            (10)
  backtest/              replay engine, cost model, statistics                          (11)
  scrape.rs              runs the embedded Python scraper as a background job (progress, cancel)
  web.rs + web_ui.html   local browser interface (std-only HTTP server + one-file page)
  main.rs                CLI (no arguments = browser interface)
tests/pipeline.rs        end-to-end tests on the synthetic market
tools/                   scrape_companies.py (stdlib-only scraper) + its offline unit tests
```
How it works
Levels. Swing highs/lows (5 bars each side, at least 1 ATR of prominence) are clustered into zones whose
width is `1 ATR` and padding `0.1 ATR`, so zones adapt to each stock's volatility. Pivots are only used once
confirmed (5 bars later) and are weighted by recency (126-session half-life) and volume. The same zone acts as
resistance when approached from below and support from above, so flipped levels are handled naturally.
Rigidity (0-100), separately for support and resistance. Six components in `[0, 1]`:
component	default weight	meaning
frequency	25%	recency- and quality-weighted test count, saturating
rejection	25%	mean ATR distance price retreated after tests (un-rejected tests dilute it)
duration	10%	span between first and last meaningful test, saturating
recency	15%	how recently the level was last meaningfully tested
volume	15%	whether tests happened on heavy RVOL
compression	10%	successive reactions getting shallower (pressure building)
A test earns credit only in proportion to the rejection that followed (below 0.5 ATR = nothing), and
frequency/duration saturate, so a level is not "strong" merely because it is old or was touched often.
Detection. For each of the last 40 sessions the zones as known before that bar are rebuilt (no
look-ahead). A breakout is the first close beyond an eligible zone (established, rigidity >= 30, >= 2
meaningful tests) by `0.25 ATR`, with RVOL >= 1.5 and Z >= 1, supportive RSI (past 50 or rising) and
Bollinger expansion or a close outside the band. The cooldown suppresses a re-break of the same level only.
Classification (first match wins; all thresholds in `[classification]`):
FAILED - a later close returned inside the zone.
EXTENDED - >= 3 ATR beyond the level, or older than 15 sessions (unless a continuation setup exists,
which is DEVELOPING and flagged `is_continuation`).
FADING - 2 of 3 symptoms (RSI rolling over, thin volume, pullback from peak), or neither momentum nor
volume still supports it.
FRESH - age <= 5, <= 1.5 ATR extended, momentum and volume supportive.
DEVELOPING - everything else that survived.
Only FRESH and DEVELOPING reach the ranking; a high score can never override this, and fewer than ten
results are returned rather than padding the list.
Score = `100 * sum(weight * component)` (rigidity 30, volume 25, RSI 15, Bollinger 15, magnitude 15), each
component capped/saturated to `[0, 1]`, plus explicit point adjustments: trend alignment (+3 / -8), consolidation
bonus (up to +6), extension (-4 per ATR beyond 1), age (-1 per session beyond 5), contradicting momentum (-6),
weak follow-through (-3), continuation (-3). Every adjustment is stored in the JSON output.
Backtest. The same `Analyzer` is replayed each simulated day on only the trailing 504 sessions. Signals fire
after the close; trades enter at the next open and exit at the close `h` sessions later (5/10/20), net of
commission, spread, slippage and short borrow. Each breakout is recorded the first time it is seen in each
classification, so FRESH (first-day) entries can be compared with EXTENDED/FADING (late) entries. Reports hit
rates, mean/median net returns, score-bucket monotonicity, Spearman IC of the composite and of each component,
indicator pairs, the daily top-N list as traded, and false-breakout frequency.
Interpretation choices worth knowing about
Age counts completed sessions since the breakout bar: 0 = the latest completed session. The spec's
"1-5 sessions" favourable window is implemented as `age <= 5`.
EXTENDED covers both price and time (stale) extension; `reason` says which.
Extension is measured in pre-breakout ATR, so the yardstick does not inflate after a big breakout bar.
The table's `ATR` column is the current ATR.
Volume score blends the breakout bar (65%) with post-breakout follow-through (35%); the table's RVOL and Z
are for the latest session, and breakout-day values are in the JSON (`breakout_rvol`, `breakout_volume_z`).
Prices are back-adjusted. FMP's `dividend-adjusted` endpoint supplies split- and dividend-adjusted OHLC.
The registry is updated only by live scans, not by `--as-of` replays.
Limitations
FMP's historical constituent list lets the backtester reconstruct membership, but delisted tickers are often
absent from the price feed, leaving some survivorship bias (logged as a warning).
Consecutive daily observations overlap, so backtest t-statistics are indicative only.
No holiday calendar: the S&P 500 index series is used as the trading calendar.
Parameters are sensible starting points, not tuned. Judge them by monotonic score-bucket performance and
positive information coefficients out of sample, not by how good one run looks.
Verification
No compiler was available when this was written. Besides hand review, the synthetic generator and the
detect -> classify pipeline were ported to Python to confirm that planted breakouts are detected with
rigidity ~70-80 and classified FRESH (age 0-1) or DEVELOPING (age 2-4). That check exposed (and led to the fix
for) a bug where a minor zone cleared a few sessions earlier suppressed the real breakout through the cooldown.
`cargo test` runs the unit tests plus `tests/pipeline.rs`, which checks the FRESH/DEVELOPING gate, causality
(no look-ahead), countertrend flagging, as-of replays and incomplete-session handling.
