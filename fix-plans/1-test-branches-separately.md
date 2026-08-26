# Plan 1 — Test each branch separately against a live Quicken database

Goal: prove `fix/raw-query-hardening` and `fix/sanitize-error-paths` each
work against a real Quicken file, independently. Plan 2 tests them together.

Expect 30–45 minutes, most of it waiting on `npm ci`.

---

## Step 0 — Locate your Quicken database (do not skip)

**The plan does not assume where your file is.** Auto-detection only looks in
`~/Documents`, and on this setup there is no `.quicken` bundle there. You must
find your file and set `QUICKEN_DB_PATH` explicitly.

Try these, in order, until one prints a path:

```bash
ls -d ~/Documents/*.quicken 2>/dev/null
mdfind -name '.quicken' 2>/dev/null | grep '\.quicken$'
ls -d /Volumes/*/*.quicken /Volumes/*/*/*.quicken 2>/dev/null
find "$HOME" -maxdepth 4 -name '*.quicken' -prune -print 2>/dev/null
```

If none of them find it, open Quicken and use **File → Show in Finder** (or
check the recent-files list) to see where the open file lives.

A `.quicken` file is a **bundle** (a directory). The database the tests read
is the `data` file inside it:

```
/wherever/My Finances.quicken/data
                             ^^^^ this is what QUICKEN_DB_PATH points to
```

### Work on a copy

Do not point the test run at your only copy. The tests open the database
read-only, but Quicken itself will be opening and possibly upgrading whatever
file you hand it.

```bash
# 1. Quit Quicken first, so the bundle is not mid-write.
QUICKEN_SRC="/full/path/to/My Finances.quicken"     # <- edit this
cp -R "$QUICKEN_SRC" "$HOME/quicken-test-copy.quicken"

# 2. Launch Quicken and open the COPY:  File → Open → ~/quicken-test-copy.quicken
#    Leave Quicken running for the whole test session. Closing it re-encrypts
#    the database and every live suite will fail.

export QUICKEN_DB_PATH="$HOME/quicken-test-copy.quicken/data"
```

### Verify before running anything

```bash
ls -l "$QUICKEN_DB_PATH"          # should exist and be large, not a few KB
sqlite3 "$QUICKEN_DB_PATH" ".tables" | tr ' ' '\n' | grep -c ZACCOUNT
```

A tiny `data` file, or `.tables` printing nothing, means the database is still
encrypted — Quicken is not running, or is not holding *this* copy open.

---

## Step 1 — Get the repository

The repo is public, so no SSH key or GitHub login is needed:

```bash
git clone https://github.com/apisani1/quicken-mac-mcp.git
cd quicken-mac-mcp
node -v        # must be >= 22
```

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
npm ci
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

### Extra manual check: the new shared result bounds

This branch added a 5000-row / 2 MB cap to *every* tool, and a real file is
the first chance to see whether any legitimate query trips it:

```bash
npx tsx src/index.ts list_categories | tail -5
npx tsx src/index.ts spending_over_time --start_date 2000-01-01 \
    --end_date 2026-12-31 --group_by_category true | tail -5
```

The second one is the intended stress case: one row per month per category
over your full history. If it errors with `returned too many rows`, that is
the cap working — but note the number and tell me, because it would mean the
5000 ceiling is too low for a real long-history file and should be raised.

---

## Step 4 — Test `fix/sanitize-error-paths`

```bash
git checkout fix/sanitize-error-paths
npm ci
npm run build
npm run lint
npm test 2>&1 | tee ~/results-sanitize.txt
```

**Baseline without a live database:** `88 passed | 123 skipped (211)`.
With the live database, `skipped` should again drop to **0**.

### Extra manual check: does a real path actually get redacted?

This is the whole point of the branch, and it is the one thing the unit tests
cannot fully prove, because they use synthetic paths. Force a real error and
look at the message:

```bash
# Point at a nonexistent file inside your real directory, so the error message
# contains a genuine path with your real folder names in it.
QUICKEN_DB_PATH="$HOME/quicken-test-copy.quicken/nonexistent" \
  npx tsx src/index.ts list_accounts 2>&1 | tail -5
```

Read the output carefully. **It must not contain your home directory, your
username, or any folder name from the path.** You should see `<path>` or `~`
instead. If any fragment of a real path survives, copy the exact output — that
is a live leak the synthetic tests missed, and it needs fixing before the PR.

---

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
