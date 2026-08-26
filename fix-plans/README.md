# fix-plans

Working plans for finishing the security-review follow-up on
`quicken-mac-mcp`. Use them **in numbered order**.

| # | Plan | Where | What it gets you |
|---|------|-------|------------------|
| 0 | [0-prerequisites.md](0-prerequisites.md) | Secure Space (admin + test account) | A non-admin `quicken-test` account, the repo, and the native module ready |
| 1 | [1-test-branches-separately.md](1-test-branches-separately.md) | `quicken-test` | Each fix branch validated against a real Quicken database, on its own |
| 2 | [2-test-combined-branch.md](2-test-combined-branch.md) | `quicken-test` | A throwaway merge of both branches, tested together, then deleted |
| 3 | [3-open-pull-requests.md](3-open-pull-requests.md) | `quicken-test` (browser) or main volume | The two PRs opened, with descriptions and review notes |
| 4 | [4-tokenizer-and-statement-policy.md](4-tokenizer-and-statement-policy.md) | Either | The two deliberately-deferred findings addressed |

Plans 0–3 are sequential. They run on the **Secure Space** volume, inside a
dedicated non-admin `quicken-test` account, so that `npm` install scripts and
the test suite never run with administrator privileges — see the threat model
in plan 0. Plan 3 needs no database and can be done from a browser in that
account, or from the main volume; plan 0 step 0a explains why that choice
matters. Plan 4 is independent design work and can happen any time after plan
3 — it is a follow-up PR, not a blocker.

## The branches

Two branches are PR-ready. Both merge into `main` cleanly and are
independent of each other:

- **`fix/raw-query-hardening`** — all four `raw_query` findings plus shared
  result bounds. 12 commits ahead of `main`.
- **`fix/sanitize-error-paths`** — the error-message path-leak finding.
  3 commits ahead of `main`.

Three branches are **superseded** and must not be used for PRs. Every one of
their commits is an ancestor of `fix/raw-query-hardening`, and they conflict
with each other:

- `fix/raw-query-limit-bypass`
- `fix/raw-query-pragma-bypass`
- `fix/raw-query-timeout`

## Why a live-database run matters

The synthetic test suite now covers the security boundary and runs
everywhere. What it cannot cover is the real Quicken schema and real data
volume. Six tests stay gated behind `QUICKEN_DB_PATH`, and none of this work
has ever run against an actual Quicken file.

The test harness fails loudly rather than quietly here: `npm test` runs
`scripts/report-live-test-status.mjs` first, and if `QUICKEN_DB_PATH` is set
but unusable it exits **1** instead of skipping. A green run with the
variable set means the live suites really ran.

## Requirements on the machine you test from

- **Node ≥ 22** (`package.json` engines). Verified working on Node 24.15.0 / npm 11.12.1.
- **Quicken for Mac, running, with the database open.** Quicken keeps its
  database encrypted at rest and replaces it with a small stub when closed.
  If Quicken is not running, every live suite fails the decryption check.
- **A copy of your Quicken file** — see the "locate your database" step in
  plan 1. Do not point the tests at your only copy.
