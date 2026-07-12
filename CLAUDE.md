# Vault conventions (schema layer)

This vault IS a GitHub-backed Obsidian vault and the **source of truth** for a Finviz→
TradingView pipeline. n8n only orchestrates; every durable byte lives here as plain text.
This file is the `raw / wiki / schema` pattern's **schema** layer — the conventions an
agent reads before touching anything. Read it first.

## The one rule that governs everything
Durable state and curation live in the repo; n8n only orchestrates. The compute layer is
swappable without losing data.

## Layers
- `config/` — the registry (`screeners.json`), the single source of screener names.
- `raw/` — **immutable** dated snapshots. Never edited, never regenerated (Finviz only
  gives today's screen). Full 25-col schema captured from day one.
- `watchlists/` — the **only stateful layer**. Per-screener rolling aggregate + keep-list.
- `wiki/` — DEFERRED. AI analysis (dated, disposable notes + append-only `log.md`).
- `sources/` — DEFERRED. Capture-only trader/news knowledge base.

## Timeseries, not documents (critical)
The signal is the **diff across dated snapshots** — a computation, not entity-page synthesis.
- Do NOT build per-ticker pages that get overwritten daily. That destroys the timeseries.
- Analysis notes are dated and disposable; they don't rot, so there is no anti-drift linter.
- `wiki/log.md` is append-only, for interpretive continuity ("energy rotation, week 2").

## Naming
- Dates: `YYYY-MM-DD` everywhere (sortable, greppable).
- `<screener>` dir = the registry `name`, **slugified**: lowercase → each run of non-`[a-z0-9]`
  → single `-` → trim. Computed from `name`, never stored; workflow asserts slugs unique at
  load. (e.g. `IPO < 1Y`→`ipo-1y`, `52w Highs`→`52w-highs`.) Full rule + rename caveat in repo.md.

## Directory tree
```
/
├── CLAUDE.md                       # this file
├── config/screeners.json           # registry: [{name, f, t, o}]
├── raw/<screener>/<year>/YYYY-MM-DD.csv  # immutable snapshot, native Finviz CSV as-is
├── watchlists/
│   └── <screener>/
│       ├── keep.md                 # checkbox keep-list + reset flag
│       ├── aggregate.txt           # rolling cumulative watchlist (TradingView)
│       └── daily/YYYY-MM-DD.txt      # that day's single-screener watchlist
├── wiki/  (DEFERRED)               # index.md, log.md, YYYY-MM-DD.md
└── sources/ (DEFERRED)            # traders/<name>/…, news/<ticker>/…
```

## Registry — `config/screeners.json`
Array of `{name, f, t, o}` (+ optional `c`). **No full URLs, no token.** The workflow assembles
the CSV export endpoint + `c=` columns + `f`/`t`/`o` + token (n8n credential).
- `f` = filter string (from the Finviz preset). `t` = fixed basket (Group Themes-style).
- `o` = sort order. A screener uses `f` OR `t`, not both.
- `c` = optional column override. Omitted → uniform 25-col company set. Present → verbatim.
  `c: ""` = pending (not runnable). Used only by the ETF basket (Group Themes) — see below.

## Export schema — 25 columns (SETTLED)
Uniform for all screeners via `c=` (not `v=`; `v=151` is only the view mode):
```
c=1,129,2,3,4,6,65,66,67,63,64,42,43,44,45,47,46,50,17,22,23,20,29,28,30
```
Ticker, Exchange, Company, Sector, Industry, Market Cap, Price, Change, Volume,
Average Volume, Relative Volume, Perf (Week, Month, Quarter, Half-Year, YTD, Year),
Volatility (Week), EPS Growth (This-Yr, QoQ, Next-5Y), Sales Growth QoQ,
Institutional (Transactions, Ownership), Short Float.

Snapshots store all 25. Watchlists use only Ticker + Exchange.

**ETF basket exception (Group Themes):** ETFs have no company fundamentals, so this one
overrides `c=` with an ETF set — Ticker, Exchange, Price, Change, Volume, Avg Vol, Rel Vol,
Perf (W/M/Q/HY/YTD/Y), Sector/Theme, Net Flows % (1M/3M/YTD/1Y). Net Flows is the rotation
signal (per-day unrecoverable → must capture). Snapshot schema is thus per-screener, read
from the CSV header row. `c=1,65,66,67,63,64,42,43,44,45,47,46,104,113,115,117,119,129`. See repo.md.

## Snapshot — `raw/<screener>/<year>/YYYY-MM-DD.csv` (immutable)
**Native Finviz CSV, written as-is** — header row + data rows, no frontmatter/markdown.
`.csv` (not `.md`) so Obsidian doesn't index thousands of files; year-partitioned likewise.
Column set (25-col company default or ETF override) is identified by the CSV header row.
Per-file status is **derived, not stored**: data rows = ok; header-only (0 rows) = empty
(still success); no file = failed/skipped.

## Watchlist `.txt` (TradingView)
One file **per screener**, comma-separated `EXCHANGE:TICKER`. Optional leading
`###<screener>` header names the list on import. **No `ALL.txt`** — screeners stay separate.
Aggregate is **cumulative-until-reset**: daily runs union today's hits in; they never
auto-drop. Bounding happens only via the manual keep-list reset.

## Keep-list — `watchlists/<screener>/keep.md`
```yaml
---
screener: "9M"
reset: false          # true → next run flushes aggregate to keepers, then self-clears
---
- [x] NASDAQ:NVDA     # [x] = keep
- [ ] NASDAQ:SMCI
```
Rule: the workflow **appends** new `- [ ]` candidate lines only; it must **never rewrite
existing lines** (keeps Git merges clean when edited mid-run). Only reset consults this file.

## sources/ frontmatter (DEFERRED — schema locked, capture-only)
```yaml
---
source: "x.com/..."   # url
author: "<handle>"
date: 2026-07-09
tickers: [NVDA, XOM]
kind: opinion         # opinion | reporting
---
```
