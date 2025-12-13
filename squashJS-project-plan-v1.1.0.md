# squashJS Project Plan — v1.1.0

## Table of Contents
- Purpose
- Current Working Approach (repeatable recipe)
- Planned Modules and Export Contracts
- Environment Variables
- Testing & Self-Test Strategy
- Editing / Compliance Rules
- README Requirements (per policy)
- API Surface (planned)
- Release & Versioning
- Open Items / Questions

## Purpose
- Capture PSA squash rankings (men and women) and tournaments (all, men, women) into repeatable JSON outputs.
- Use headless browsing to fully render dynamic tables (tab switches, rankings load-more pagination), then parse to structured data for APIs.

## Current Working Approach (repeatable recipe)
- Use Puppeteer when HTTP fetch returns empty/blocked HTML.
- Rankings: wait for `table.player-list.filterable` to render rows, click `.load-more a.button` until no new rows appear.
- Women’s rankings: click `div.tab[data-id="female"]` before the load-more loop.
- Tournaments: go to tournaments URL, click the gender filter (all/mens/womens); no load-more; capture once rendered.
- Use a temp user-data-dir and desktop UA to avoid crashpad/sandbox issues.

## Planned Modules and Export Contracts
- `fetchRankingsPage.mjs`
  - Export: `async function fetchRankingsHtml({ url, outputPath })` → `{ method, html, savedTo? }`
  - HTTP first, Puppeteer fallback; optional save; self-test writes `rankings.html`.
- `fetchRankingsWithLoadMore.mjs`
  - Export: `async function fetchRankingsWithLoadMore({ url, gender, maxClicks, outputPath })` → `{ clicks, rows, html, savedTo? }`
  - Select gender tab, wait for rows, click load-more up to `maxClicks`, save HTML; self-test asserts rows > 20.
- `parseRankings.mjs`
  - Export: `function parseRankingsFromHtml(html)` → `RankingRow[]` with `{ gender, rank, name, tournaments, points, flag, playerId, profileUrl }`
  - Tolerates arrow symbols; extracts ISO flag; infers gender from `data-filter`; self-test fails on zero rows.
- `captureRankings.mjs`
  - Export: `async function captureAllRankings({ outputDir, maxClicks })`
  - Calls load-more fetch for men and women, parses, writes HTML/JSON, and returns parsed data + paths; self-test ensures row counts > 20.
- `fetchTournamentsWithFilters.mjs`
  - Export: `async function fetchTournaments({ url, filter, outputPath })` → `{ filter, rows, html, savedTo? }`
  - Click gender filter tab, wait for rows; no load-more; save HTML; self-test ensures rows > 0.
- `parseTournaments.mjs`
  - Export: `function parseTournamentsFromHtml(html)` → `TournamentRow[]`
  - Fields: `{ orderIndex, gender, status, category, name, url, location, startDate, endDate, men: {drawSize, prize, level{raw,normalized}}, women: {…}, coverage }`; `orderIndex` preserves page order; normalizes dates using season year when possible; self-test fails on zero rows.
- `captureTournaments.mjs`
  - Export: `async function captureAllTournaments({ url, outputDir })`
  - Fetches all/mens/womens, parses (with `orderIndex`), saves HTML/JSON, returns parsed data + paths; self-test ensures rows > 0 per filter.
- `scripts/install-deps.sh`
  - Installs `puppeteer`, `cheerio`, and future deps.
- `scripts/summaryReport.mjs`
  - DB-only report showing top-10 men/women rankings plus tournaments (default min prize 200k), ordered by `order_index` then date; args: `[limit] [pivotDate] [minPrize]`.

## Environment Variables
- `PUPPETEER_EXECUTABLE_PATH` (optional): override browser binary if bundled Chromium fails.
- `PUPPETEER_HEADLESS` (optional): set `false` for debugging.
- `HTTP_PROXY` / `HTTPS_PROXY` if required.

## Testing & Self-Test Strategy
- Every module includes `if (import.meta.url === \`file://${process.argv[1]}\`)` guards.
- Capture modules: assert rows > 0; load-more asserts at least one click where applicable.
- Parse modules: assert parsed row count > 0; log a sample.
- Orchestrators: assert non-empty per gender/filter; write JSON outputs; log paths.
- `testRunner.mjs` executes all modules/orchestrators and reports pass/fail counts.

## Editing / Compliance Rules
- Follow OWP coding policy v1.0.10: package plan present; modules documented with JSDoc matching export contracts; inline self-tests guarded by `import.meta.url` check.
- Keep install script (`scripts/install-deps.sh`) current with required deps.
- README must describe purpose, install, env vars, module exports, self-tests, expected outputs.
- Use `outputs/` for orchestrated saves and `test-outputs/` for test/one-off runs; keep `node_modules` out of VCS, retain `package.json`/`package-lock.json`.

## README Requirements (per policy)
- Purpose, install steps, env var notes, module exports, self-test commands, expected outputs (paths + row counts), test runner usage.

## Database Schema / Plan
- Rankings table `<prefix>rankings`: columns capture_gender, gender, rank, name, tournaments, points, flag, player_id, profile_url, captured_at (timestamptz). Capture runs truncate then reload.
- Tournaments table `<prefix>tournaments`: columns order_index (site order), filter (all/mens/womens), gender, status, category, name, url, location, start_date, end_date, men JSONB, women JSONB, coverage JSONB, captured_at (timestamptz). Capture runs truncate then reload; keep `order_index` for downstream API sorting.
- Schema prefixes optional via `SQUASH_SCHEMA_PREFIX` / `SQUASH_TEST_SCHEMA_PREFIX`.

## API Surface (planned)
- `GET /api/rankings`
  - Params: `gender` (male|female; default could be male or both combined via capture), `limit` (int, optional).
  - Logic: use `captureAllRankings` (or fetch+parse) then filter/limit in-memory; return `{ rankings: RankingRow[] }`.
- `GET /api/tournaments`
  - Params: `gender` (all|mens|womens), `type` (diamond|gold|silver|copper|bronze mapped from level/category), `startDate`, `endDate` (inclusive).
  - Logic: use `captureAllTournaments` (or fetch+parse single filter), then filter in-memory by gender/type/date; return `{ tournaments: TournamentRow[] }` sorted by `order_index` (then start_date) and include `order_index` so clients can mirror page order.
- Endpoints should reuse orchestrator outputs to avoid duplicate fetches and expose only documented fields.

## Release & Versioning
- Target stable release v1.1.0 (API support) after README/self-tests/install script finalized.

## Open Items / Questions
- Confirm mapping of tournament `type` from level/category (use normalized level).
- Confirm default behaviour for rankings `gender` param (male only vs combined).
- Validate date normalization against season boundaries for accurate range filters.
