# AGENTS.md

## What this repo is

- The entire deliverable is `stock_dashboard.html`: a single-file Vietnam stock dashboard (valuation, dividends, fundamentals). No build system, no package.json, no in-repo tests, no CI.
- Rename history: created as `index.html`, user-renamed to `pepb_dashboard.html`, then to `stock_dashboard.html`. Old harness/test files in `%TEMP%` may reference the old names.
- `stock-dashboard.json` at the root is an exported OpenCode chat-session log (~6 MB), not app data or config — do not treat it as a data source or edit it.
- Scope: works for individual stocks AND index tickers (`INDEX_TICKERS`: VNINDEX, VN30, HNXINDEX, ...). The dashboard adapts by sector: banks (detected via `BANK_TICKERS` set fallback) get ROE/NIM/NPL/leverage charts and must NOT show ROIC/FCF/OCF/D/E/Capex/P-AFCF; non-banks get the reverse set (ROIC, Adjusted FCF, OCF, Capex, D/E plus the P/Adjusted FCF valuation metric/chart; no NIM/NPL). All fundamental charts share ONE Quarterly/Yearly toggle (default Yearly); it also drives the P/Adjusted-FCF series.
- All app code lives in ONE inline `<script>` block (the other `<script src=...>` is the Chart.js 4.4.1 CDN tag). CSS is inline in `<style>`. File must stay UTF-8 **without BOM**; UI text is Vietnamese — never re-encode or "fix" characters (em-dashes `—` are intentional; mojibake like `Ã` means you broke encoding).

## How to verify changes (no test runner exists — recreate the harness)

The HTML cannot be executed directly under node. Workflow used by prior sessions (harness files live in `%TEMP%\opencode\` and are ephemeral — recreate if missing):

1. Extract the inline script and syntax-check:
   ```powershell
   $t = [IO.File]::ReadAllText("...\stock_dashboard.html", [Text.UTF8Encoding]::new($false))
   $m = [regex]::Match($t, '(?s)<script>(.*)</script>')
   [IO.File]::WriteAllText("$env:TEMP\opencode\vnval_app.js", $m.Groups[1].Value, [Text.UTF8Encoding]::new($false))
   node --check vnval_app.js
   ```
2. Unit tests = concat `vnval_app.js` + a test file, run with node. Suites assert via `PASS:`/`FAIL:` lines; count with `Select-String`. Suites: fundamentals (~153 tests), legacy valuation (~67; its `vsSentence` asserts were rewritten to `relText` after the assessment-panel removal), P/Adjusted FCF (~59), plus a DOM-stub render smoke test and a live E2E check that hits the real APIs. The `*_body*.js` files are the canonical test sources — prebuilt `*_full.js` embeds go stale as the app evolves, always re-concat before running.
3. Structural checks after edits: exactly 2 `<script`/`</script>` tags (count the prefix `<script`, NOT the literal `<script>` — the CDN tag has attributes so the literal matches only once), 1 `<body>`, no BOM, no mojibake.
4. The script ends with `if (typeof document !== "undefined") initApp();` so node can load it — keep that guard.

Node-harness gotchas: append test code INSIDE the same `eval(code + ...)` string (top-level functions declared by eval are not visible outside it); the Chart stub signature is `(canvas, config)`; stub `document`, `localStorage`, `getComputedStyle`, `window.matchMedia`, and `fetch`.

## Live data sources (browser fetches directly; CORS matters)

- VNDirect `api-finfo.vndirect.com.vn/v4/ratios` — P/E (`PRICE_TO_EARNINGS`), P/B (`PRICE_TO_BOOK`), bank NPL (`BAD_LOANS_RATIO_AQ`, quarterly fractions). CORS-open (`*`). Also used for price quotes, company names, dividend events.
- cafef `apiweb.cafef.vn/api/v2/BCTC/GetReportCDKT` (balance sheet) and `/api/v1/BCTC/GetReportDetail` (income) + `GetReportLCTT` (cash flow). cafef only sends `Access-Control-Allow-Origin: *` when the request has an `Origin` header — browsers are fine, curl/PowerShell without Origin shows no CORS header (don't misdiagnose).
- cafef v1 endpoints are intermittently very slow (504s, >30s hangs). `fetchCafefReport` has a 15s timeout + 3 attempts with backoff, and statements are fetched with `Promise.allSettled` so one flaky endpoint doesn't kill the rest (cash flow and quarterly data are optional/partial-tolerant). Do NOT regress this to plain `Promise.all`.
- Never fabricate live values. If a feed fails, degrade gracefully (`fundUnavailable` message, demo fallback labeled via `isDemo`/`fallbackReason`/demo badge). `USE_LIVE_DATA = false` forces the synthetic mock provider.
- cafef zero-fills the parent profit line (26) for older periods → bank ROE falls back to line 21 and drops rows where both are 0. Don't "simplify" this away.
- Cash flow lines (cafef): HDKD_20 = OCF, HDDT_22 = capex, HDDT_28 = "interest, dividends and profit received". Associate/JV dividends come from HDDT_28 and are treated as NOT already in OCF (per-ticker override: `ASSOCIATE_DIVS_IN_OCF_OVERRIDES`), so Adjusted FCF = OCF − Capex + that amount — never double-count.
- Dividend history must keep cash dividends and stock dividends strictly separate (never convert stock % into VND or count it as cash).

## Domain logic invariants

- `classifyValuation` is the single source of truth for Cheap/Fair/Expensive and is intentionally asymmetric: discount >10% (`CHEAP_DISCOUNT`) → Cheap; 0–10% discount → status Fair but `type: "discount"` so the UI still shows the magnitude ("5.0% discount", not "≈ in line"); premium >5% (`FAIR_TOLERANCE`) → Expensive. Boundaries are exclusive (exactly 10%/5% stays Fair).
- `getStockValuation` has a single caller but is kept deliberately (test suite + compatibility entry point). CSS classes `tone-green/amber/orange/red` look unused but are built dynamically (`"assess-badge tone-" + a.tone`) — never delete them as dead code.
- P/Adjusted FCF (non-banks only): quarterly ratios divide period-end market cap by **TTM** Adjusted FCF (`calculateTTM` — must use `Number.isFinite`, plain `isFinite(null)` is true); yearly ratios use annual AFCF. Period market cap = P/B at period end × reported equity (≡ price × shares, historical values only — never today's price). Non-positive AFCF renders as N/M, never a negative or zero multiple. The series follows the Quarterly/Yearly toggle, NOT the 3y/5y/10y selector.
- Price Range section (`buildPriceRange`): EPS/BVPS are DERIVED as price÷current-ratio identities from the same feed (never a second source). Attractive Entry = reasonable range × (1−MoS) per spec sections #6/#27 — the spec's own #31 example contradicts this; do not "fix" the code to match it. Combined fair value weights come from `VALUATION_WEIGHTS` (bank 40/60 PE/PB, else 50/50) and invalid methods are excluded BEFORE weighting (undefined×0 = NaN trap). Non-overlapping P/E & P/B spans must render "Reference Range", never "Fair Value". Copy guard: only "historical valuation-based/reference" wording — never intrinsic/true/guaranteed value.

## Product decisions from the user (do not undo)

- No default ticker on first load; empty input must fail silently (no error alert). The "Invalid ticker format" error appears only for non-empty invalid input (guard at top of `analyze()`).
- Current P/E and P/B display with 2 decimal places everywhere they appear as "current" (hero, stats table, comparison card, range card, chart Current tooltip); averages/medians/axes stay at 1.
- Design-token locks (polish pass): radius scale is documented above `.card` (cards 14 via `--radius`, panels/status 12, containers/controls 10, buttons/inputs 9, segmented pills 8, label/badge pills 999). Light-theme status tokens are AA-tuned (`--green #047857`, `--red #cc2222`, `--orange #c2410c`, `--faint == --muted`) — do NOT lighten them back; dark `--faint` is `#75879f`. Raw hex outside token blocks is limited to: always-dark topbar/loading whites, the brand-mark gradient, and the gauge heat-scale mid-stop. Interactive controls have a global `:focus-visible` ring; decorative transitions collapse under `prefers-reduced-motion` (the loading spinner is exempt as essential feedback).
