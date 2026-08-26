# Plan 2 — Test both branches combined, on a throwaway branch

Do this **after** plan 1 passes for both branches.

Goal: catch anything that only breaks when both changes are present. The
branch is disposable — it exists to be tested and deleted, and nothing is ever
pushed from it.

Expect 15 minutes.

---

## Why this step exists at all

The two branches touch disjoint files (`src/server.ts` versus `src/tools/`)
and merge without conflict, so the risk is low. But there is one real place
they meet:

`raw_query` errors and the new `limits.ts` errors both travel through
`formatToolError()` in `src/server.ts` — which is precisely the function
`fix/sanitize-error-paths` rewrote. A SQLite error raised inside the
`raw_query` **child process** can carry the database path in its message, and
that message crosses the IPC boundary before it reaches the sanitizer.

Neither branch tests that path, because neither branch contains both halves.
That is the specific thing this plan checks.

---

## Step 0 — Prerequisites

Same live database setup as plan 1. If you are in a new shell:

```bash
export QUICKEN_DB_PATH="$HOME/quicken-test-copy.quicken/data"
node scripts/report-live-test-status.mjs; echo "exit=$?"   # must be exit=0
```

Quicken must still be running with the copy open.

---

## Step 1 — Build the throwaway branch

```bash
cd quicken-mac-mcp
git fetch origin
git checkout -b tmp/live-verify origin/fix/raw-query-hardening
git merge --no-edit origin/fix/sanitize-error-paths
```

This merge is **expected to be clean** — it was verified conflict-free. If git
reports a conflict, stop and tell me: it means one of the branches moved since
this plan was written, and the resolution is a decision, not a mechanical fix.

---

## Step 2 — Full verification

```bash
npm ci
npm run build
npm run lint
npm test 2>&1 | tee ~/results-combined.txt
```

**Baseline without a live database:** `130 passed | 109 skipped (239)`.
That is more than either branch alone — 116 + 88 minus the tests they share —
and it is the number to sanity-check against.

**With the live database, expect all 239 to run: `skipped` should be 0.**

---

## Step 3 — The interaction check (the reason for this plan)

Force a `raw_query` failure whose underlying SQLite error will contain the
database path, and confirm the sanitizer catches it *after* it crosses the
child-process boundary:

```bash
# A syntax error raised inside the child process, propagated to the parent.
npx tsx src/index.ts raw_query --sql "SELECT * FROM no_such_table_xyz" 2>&1 | tail -5

# A query that trips the shared 2 MB response cap, if your file is large enough.
npx tsx src/index.ts raw_query \
  --sql "SELECT * FROM ZTRANSACTION" 2>&1 | tail -5

# An unreadable database, so the failure happens at open time.
QUICKEN_DB_PATH="$HOME/quicken-test-copy.quicken/nonexistent" \
  npx tsx src/index.ts raw_query --sql "SELECT 1" 2>&1 | tail -5
```

For every one of these, check the output for:

1. **No real path fragments** — no `/Users/<you>`, no real folder names, no
   `.quicken` bundle name. `<path>` and `~` are the expected redactions.
2. **The error is still useful** — over-redaction is its own failure. If the
   message has been reduced to something like `Error: <path>` with no
   indication of what went wrong, note it: the sanitizer is too aggressive and
   that is worth fixing before the PR.

Also confirm the ordinary path still works end to end:

```bash
npx tsx src/index.ts raw_query --sql "SELECT COUNT(*) as cnt FROM ZACCOUNT"
npx tsx src/index.ts raw_query --sql "SELECT COUNT(*) as cnt FROM ZACCOUNT; -- comment"
```

The second is the trailing-semicolon-plus-comment case that was broken until
recently; it should return a count, not a syntax error.

---

## Step 4 — Tear down

The combined branch has done its job. Do not push it, and do not open a PR
from it — the two PRs come from the individual branches, per plan 3.

```bash
git checkout main
git branch -D tmp/live-verify
```

---

## Step 5 — Report back

Keep `~/results-combined.txt` and record:

- the final `Tests` line (should be `239 passed | 0 skipped`)
- the exact output of each command in step 3
- anything that passed individually in plan 1 but fails here — that is the
  interaction this whole plan was built to find, and it would need fixing
  before either PR merges

If this is green, go to **plan 3** and open the PRs.
