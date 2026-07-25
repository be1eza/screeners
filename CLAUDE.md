# Vault conventions (schema layer)

This vault IS a GitHub-backed Obsidian vault and the **source of truth** for a Finviz→
TradingView pipeline. n8n only orchestrates; every durable byte lives here as plain text.
This file is the `raw / wiki / schema` pattern's **schema** layer — the conventions an
agent reads before touching anything. Read it first.

## The one rule that governs everything
Durable state and curation live in the repo; n8n only orchestrates. The compute layer is
swappable without losing data.

## Layers
- `config/` — the registry (`screeners.json`, single source of screener names) +
  `columns.json` (named export column sets). Both read from the repo at every run start.
- `raw/` — **immutable** dated snapshots. Never edited, never regenerated (Finviz only
  gives today's screen). Full schema captured from day one (28-col company / 21-col basket).
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
├── config/columns.json             # named column sets: {company, basket}
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
- `c` = optional column-set **name**, resolved against `config/columns.json`. Omitted → the
  `company` set (all 13 company screeners). `"basket"` → the ETF set (all 3 baskets). A
  digit-leading string = a literal column list (escape hatch). `""` = pending (not runnable).
  An unknown name **throws at load** — never silently passed through.
- `watchlist` + `aggregate` = two output gates, **present in every object** (`true`/`false`/`null`).
  `watchlist: false` → snapshot-only, no ticker list (the ETF baskets). `aggregate: false`
  → daily watchlist only, no rolling aggregate (daily-observation feeds: 5 Days Up/Down 20%, 52w Highs). `aggregate:
  null` → not applicable (set when `watchlist: false`). Always explicit, not absent-defaulted.

## Export schema — exactly TWO column sets (SETTLED)
Both defined in **`config/columns.json`** (not in the workflow) and applied via `c=`
(not `v=`; `v=151` is only the view mode). Both end in `52,53,54`.

**Company — 28 cols**, uniform across all 13 company screeners:
```
c=1,129,2,3,4,6,65,66,67,63,64,42,43,44,45,47,46,50,17,22,23,20,29,28,30,52,53,54
```
Ticker, Exchange, Company, Sector, Industry, Market Cap, Price, Change, Volume,
Average Volume, Relative Volume, Perf (Week, Month, Quarter, Half-Year, YTD, Year),
Volatility (Week), EPS Growth (This-Yr, QoQ, Next-5Y), Sales Growth QoQ,
Institutional (Transactions, Ownership), Short Float, SMA 20/50/200.

**ETF basket — 21 cols**, identical for **all 3 baskets** (only the `t` ticker list differs).
ETFs have no company fundamentals, so they override `c=`:
```
c=1,129,65,66,67,63,64,42,43,44,45,47,46,104,113,115,117,119,52,53,54
```
Ticker, Exchange, Price, Change, Volume, Avg Vol, Rel Vol, Perf (W/M/Q/HY/YTD/Y),
Sector/Theme, Net Flows % (1M/3M/YTD/1Y), SMA 20/50/200. Net Flows is the rotation signal.
The 3 baskets are a **rotation hierarchy**: `Markets` (asset class / index regime) →
`Sectors` (all 11 GICS sectors) → `Group Themes` (narrow thematic). Same columns ⇒ directly comparable.

Watchlists use only Ticker + Exchange.

**Reading SMA cols:** `52,53,54` are the **% distance from price to that SMA**, not the SMA level
(`-0.81%` = 0.81% below its 20-day). Trend position, not price.

**Two rules that must hold:**
- Exchange (`129`) sits 2nd in every set so the header starts `"Ticker","Exchange"` — **every `c`
  must begin `1,129`** or header-validation silently rejects the snapshot.
- **Extend a set at the tail; never reorder or remove.** Snapshots are immutable, so a set change
  is permanent history. **Files before 2026-07-25 lack the SMA columns** (25-col company /
  18-col Group Themes) — read by header *name* and tolerate absent tail columns; never assume width.

## Snapshot — `raw/<screener>/<year>/YYYY-MM-DD.csv` (immutable)
**Native Finviz CSV, written as-is** — header row + data rows, no frontmatter/markdown.
`.csv` (not `.md`) so Obsidian doesn't index thousands of files; year-partitioned likewise.
Column set (28-col company default or 21-col ETF override) is identified by the CSV header row.
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
