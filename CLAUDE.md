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
- `<screener>` dir = the registry `name`, slugified: **lowercase, spaces→`-`**, kept
  reversible against the registry. (Slug rule per repo.md; confirm at build if edge cases.)

## Directory tree
```
/
├── CLAUDE.md                       # this file
├── config/screeners.json           # registry: [{name, f, t, o}]
├── raw/<screener>/YYYY-MM-DD.md     # immutable snapshot, full schema
├── watchlists/
│   ├── <screener>/
│   │   ├── keep.md                 # checkbox keep-list + reset flag
│   │   ├── aggregate.txt           # rolling cumulative watchlist (TradingView)
│   │   └── daily/YYYY-MM-DD.txt      # that day's single-screener watchlist
│   └── ALL.txt                     # every screener as ### sections (one import)
├── wiki/  (DEFERRED)               # index.md, log.md, YYYY-MM-DD.md
└── sources/ (DEFERRED)            # traders/<name>/…, news/<ticker>/…
```

## Registry — `config/screeners.json`
Array of `{name, f, t, o}`. **No full URLs, no token.** The workflow assembles the CSV
export endpoint + uniform `c=` columns + `f`/`t`/`o` + token (n8n credential).
- `f` = filter string (from the Finviz preset). `t` = fixed basket (Group Themes-style).
- `o` = sort order. A screener uses `f` OR `t`, not both.

## Export schema — 25 columns (SETTLED)
Uniform for all screeners via `c=` (not `v=`; `v=150` is only the view mode):
```
c=1,129,2,3,4,6,65,66,67,63,64,42,43,44,45,47,46,50,17,22,23,20,29,28,30
```
Ticker, Exchange, Company, Sector, Industry, Market Cap, Price, Change, Volume,
Average Volume, Relative Volume, Perf (Week, Month, Quarter, Half-Year, YTD, Year),
Volatility (Week), EPS Growth (This-Yr, QoQ, Next-5Y), Sales Growth QoQ,
Institutional (Transactions, Ownership), Short Float.

Snapshots store all 25. Watchlists use only Ticker + Exchange.

## Snapshot — `raw/<screener>/YYYY-MM-DD.md` (immutable)
```yaml
---
screener: "9M"
date: 2026-07-09
row_count: 42
status: ok            # ok | empty | failed  (empty-but-valid is success)
---
```
Followed by the full 25-col rows. **Row storage format (markdown table vs. embedded CSV
block) is not yet settled** — do not assume; resolve at build step 1.

## Watchlist `.txt` (TradingView)
Comma-separated `EXCHANGE:TICKER`. Sections: `###SectionName,SYM,SYM`.
`ALL.txt` = one `###<screener>` section per screener, concatenated → single import.
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
