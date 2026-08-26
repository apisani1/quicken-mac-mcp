# Plan 1 — Test each branch separately against a live Quicken database

Goal: prove `fix/raw-query-hardening` and `fix/sanitize-error-paths` each
work against a real Quicken file, independently. Plan 2 tests them together.

**Where this runs:** the `quicken-test` standard account on the Secure Space
volume, set up in plan 0. Everything below assumes you are logged into that
account with Quicken running and the synthetic file open.

Expect 30–45 minutes, most of it waiting on `npm ci`.

---

## Step 0 — Confirm the database path (do not skip)

Plan 0 step 3 built a synthetic Quicken file in the test account. Confirm it:

```bash
ls -d ~/Documents/*.quicken
export QUICKEN_DB_PATH="$HOME/Documents/My Test Finances (2026).quicken/data"

# 1. Is it a readable database with the expected schema?
sqlite3 -readonly "$QUICKEN_DB_PATH" ".tables" | tr ' ' '\n' | grep -x ZACCOUNT

# 2. Does it hold data?
sqlite3 -readonly "$QUICKEN_DB_PATH" "SELECT COUNT(*) FROM ZACCOUNT;"
sqlite3 -readonly "$QUICKEN_DB_PATH" "SELECT COUNT(*) FROM ZTRANSACTION;"
```

Read the two results differently — they diagnose different problems:

- **Step 1 errors** (`file is not a database`, `no such table`, or no output):
  the file is still the encrypted stub. Quicken is not running, or is not
  holding *this* file open.
- **Step 1 succeeds but a count is 0:** the database is decrypted and readable;
  the table is simply empty. That is a data-population problem — go back to
  plan 0 step 3, not a Quicken-is-closed problem.

Quote the path everywhere: it contains spaces, which is deliberate (see plan 0).

### Two reminders from plan 0

- Quicken must be **running in the `quicken-test` session** with the synthetic
  file open. Closed Quicken means an encrypted stub and every live suite fails.
- Never point `QUICKEN_DB_PATH` at your real Quicken file. The tests open the
  database read-only, but Quicken will open and possibly upgrade whatever file
  you hand it, and the point of the synthetic file is that no real data is
  reachable from this account at all.

### Verify before running anything

```bash
ls -l "$QUICKEN_DB_PATH"          # should exist and be large, not a few KB
sqlite3 "$QUICKEN_DB_PATH" ".tables" | tr ' ' '\n' | grep -c ZACCOUNT
```

A tiny `data` file, or `.tables` printing nothing, means the database is still
encrypted — Quicken is not running, or is not holding *this* copy open.

---

## Step 1 — Refresh the repository

Plan 0 step 4 already cloned it and installed dependencies. Do **not** clone
again — the directory exists, and a second `npm ci` would re-run every
dependency install script for nothing.

```bash
cd ~/quicken-mac-mcp
git fetch origin
node -v        # must be >= 22
```

Both fix branches carry identical `package.json` and `package-lock.json`, so
the single install from plan 0 covers every branch you check out below. Re-run
`npm ci` only if you switch Node versions or delete `node_modules/`.

---

## Step 2 — Preflight the database wiring

From the repo, with `QUICKEN_DB_PATH` exported:

```bash
npm ci
node scripts/report-live-test-status.mjs; echo "exit=$?"
```

- Success: `[live-db] ENABLED via QUICKEN_DB_PATH; live suites are required.` and `exit=0`
- Failure: `[live-db] ERROR: QUICKEN_DB_PATH is set, but …` and `exit=1`

Do not continue until this passes. Every later failure would otherwise be
ambiguous between "the code is broken" and "the database was never readable".

---

## Step 3 — Test `fix/raw-query-hardening`

```bash
git checkout fix/raw-query-hardening
npm run build
npm run lint
npm test 2>&1 | tee ~/results-hardening.txt
```

**Baseline without a live database:** `116 passed | 109 skipped (225)`.

**With a live database, expect all 225 to run.** The number that matters is
`skipped` — it should drop to **0**. If it is still 109, `QUICKEN_DB_PATH` is
not reaching vitest (check that you exported it in *this* shell).

What the live run adds that the synthetic suite cannot:

- `raw_query` against the real Quicken schema — smoke `SELECT`, the 500-row
  cap on a real `ZTRANSACTION`, subqueries and joins
- every curated tool against real data (accounts, categories, transactions,
  spending, portfolio)
- cross-tool consistency checks (spending totals reconciling across tools)

### Extra manual check: does the shared cap fire on ordinary queries?

This branch added a 5000-row / 2 MB bound to every tool. Confirm it does not
trip on normal use of the synthetic file:

```bash
npx tsx src/index.ts list_categories | tail -3
npx tsx src/index.ts spending_over_time --start_date 2000-01-01 \
    --end_date 2026-12-31 --group_by_category true | tail -3
```

Whether that ceiling is high enough for a **real** long-history file is a
separate question, and it is deliberately not asked here — nothing in this
account should touch real data. Plan 0 has it as an optional read-only check
you run from the administrator account.

---

## Step 4 — Test `fix/sanitize-error-paths`

```bash
git checkout fix/sanitize-error-paths
npm run build
npm run lint
npm test 2>&1 | tee ~/results-sanitize.txt
```

**Baseline without a live database:** `88 passed | 123 skipped (211)`.
With the live database, `skipped` should again drop to **0**.

### Extra manual check: does a real path actually get redacted?

This is the whole point of the branch, and the one thing the unit tests cannot
fully prove, because they use invented paths. Here the path is a real one from
a live filesystem — and the bundle name from plan 0 has the spaces and
parentheses that defeated the old sanitizer. Force an error and read the
message:

```bash
# A nonexistent file inside the synthetic bundle, so the error carries a real
# path containing "My Test Finances (2026).quicken".
QUICKEN_DB_PATH="$HOME/Documents/My Test Finances (2026).quicken/nonexistent" \
  npx tsx src/index.ts list_accounts 2>&1 | tail -5
```

Read the output carefully. **It must not contain `/Users/quicken-test`, nor
any fragment of `My Test Finances (2026).quicken`.** You should see `<path>`
or `~` instead. Output like `<path> Test Finances (2026).quicken` is the exact
punctuation leak this branch fixes reappearing — copy it verbatim and report
it, because the synthetic tests missed it.

---

## Sparse-data failures: triage before reporting a bug

46 assertions across 18 live suites expect non-empty results. If your synthetic
file is missing a data shape, those tests fail — and the failure looks like a
code defect when it is a data gap.

Before treating any live failure as real, check the shape it needs:

```bash
sqlite3 "$QUICKEN_DB_PATH" "SELECT ZTYPENAME, COUNT(*) FROM ZACCOUNT GROUP BY ZTYPENAME;"
sqlite3 "$QUICKEN_DB_PATH" "SELECT COUNT(*) FROM ZTRANSACTION;"
sqlite3 "$QUICKEN_DB_PATH" "SELECT COUNT(*) FROM ZLOT;"          -- 0 => list_portfolio will fail
sqlite3 "$QUICKEN_DB_PATH" "SELECT COUNT(*) FROM ZSECURITYQUOTE;" -- 0 => quote enrichment will fail
```

A failure is a **data gap** if the assertion is `toBeGreaterThan(0)` or
`length > 0` on a table your file does not populate. It is a **real failure**
if the tool returns rows but the values, ordering, or totals are wrong — those
are the assertions worth acting on.

Record which is which when you report back. A run where `list_portfolio` fails
for want of a brokerage account is still a useful run; it just does not cover
portfolio code.

## Step 5 — Report back

Keep both result files (`~/results-hardening.txt`, `~/results-sanitize.txt`).
For each branch record:

- the final `Tests` line (passed / skipped / total)
- any failing test names with their assertion output
- the output of the two manual checks above

If both branches are green with `0 skipped`, go to **plan 2**.

If something fails, stop and capture it. A live-only failure is genuinely
interesting: it means the real schema or real data volume does something the
synthetic database does not, which is exactly what this run exists to find.
