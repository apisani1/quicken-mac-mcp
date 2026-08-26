# Plan 3 — Open the two pull requests

Do this **after** plans 1 and 2 are green.

Two PRs, one per branch. They are independent and can land in either order.

---

## Step 0a — Where to run this

Run plan 3 from your **normal working volume**, with your usual account and an
authenticated `gh`. Nothing here touches a Quicken database, so none of the
isolation from plans 0–2 applies, and no privilege restrictions are assumed.

One carry-over from that setup: do not authenticate `gh` inside the
`quicken-test` account. A token with write access to your repositories does
not belong in the account where untrusted `npm` install scripts ran.

If you ever do want to open these PRs from a machine or account without `gh`,
the branches are already pushed, so a browser works just as well:

- https://github.com/dweekly/quicken-mac-mcp/compare/main...apisani1:quicken-mac-mcp:fix/raw-query-hardening?expand=1
- https://github.com/dweekly/quicken-mac-mcp/compare/main...apisani1:quicken-mac-mcp:fix/sanitize-error-paths?expand=1

## Step 0 — Decide the target repository

This checkout has two remotes:

| Remote | Repo | Role |
|--------|------|------|
| `origin` | `apisani1/quicken-mac-mcp` | your fork |
| `upstream` | `dweekly/quicken-mac-mcp` | the project these fixes belong to |

**Recommended: open both PRs against `dweekly:main`.** These are fixes to the
upstream project, and sending them there is what gets them reviewed by the
maintainer.

### One thing to handle first if you target upstream

`fix/raw-query-hardening` carries `SECURITY_AUDIT_2026-08-25.md`, which exists
on your fork's `main` but not upstream. It would show up as a new file in the
PR diff (+107 lines). Decide deliberately:

- **Include it** — it documents where every fix came from, which is genuinely
  useful review context. Mention it in the PR description so it does not look
  accidental.
- **Exclude it** — keep the PR to code only:

  ```bash
  git checkout fix/raw-query-hardening
  git rm SECURITY_AUDIT_2026-08-25.md
  git commit -m "chore: keep the local audit report out of the upstream PR"
  git push origin fix/raw-query-hardening
  ```

  Because upstream never had the file, adding and then removing it nets to
  nothing in the PR diff.

The `fix-plans/` folder lives only on your fork's `main`, so it is **not** in
either PR. Do not merge `main` into a fix branch, or it will be.

`fix/sanitize-error-paths` has neither file and needs no cleanup.

---

## Step 1 — PR 1: `fix/raw-query-hardening`

Diff vs upstream: 10 files, ~1000 insertions.

```bash
gh pr create \
  --repo dweekly/quicken-mac-mcp \
  --base main \
  --head apisani1:fix/raw-query-hardening \
  --title "fix(raw_query): close row-cap, pragma, and timeout findings; bound all tool results" \
  --body-file fix-plans/pr-body-hardening.md
```

If you would rather target your own fork first, swap
`--repo dweekly/quicken-mac-mcp` for `--repo apisani1/quicken-mac-mcp`.

### Body — save as `fix-plans/pr-body-hardening.md`

```markdown
Addresses four findings from a security review of `raw_query`, plus one
generalization the review turned up. Each was independently validated by a
second reviewer (Codex) before and after the fixes.

## Findings fixed

**Row-cap bypass via nested LIMIT.** The old code searched for "the" LIMIT
clause in the raw SQL, which cannot distinguish an outer LIMIT from one nested
in a subquery — `SELECT * FROM (SELECT * FROM t LIMIT 100000) x` clamped the
inner limit and never added an outer bound. The caller's query is now wrapped
as a subquery under a single outer `LIMIT 500`, so the final row count is
bounded regardless of what appears inside.

**`pragma_*()` function bypass.** `\bPRAGMA\b` does not match
`pragma_database_list()`, because `_` is a word character, so SQLite's
table-valued pragma functions could disclose metadata including database file
paths. Added `PRAGMA_\w*` to the blocklist.

**Unbounded query execution.** `better-sqlite3` runs statements synchronously
in native code, and neither a graceful signal nor `Worker#terminate()` can
preempt a long-running native call. An expensive `SELECT` therefore blocked
the MCP server event loop indefinitely. Queries now run in a forked child
process with a 10s wall-clock timeout, killed with SIGKILL — the only thing
guaranteed to reclaim a process stuck in native SQLite code. Concurrency is
capped at 3 child processes, since each can burn CPU for the full timeout.

**Trailing terminator handling.** A trailing `--` comment used to swallow the
wrapper's closing tokens, and `SELECT ...; -- comment` failed with an opaque
syntax error. The terminator is now found by a single pass that tracks string
literals, quoted identifiers, and line/block comments, so `SELECT ';' as s` is
not corrupted. This is normalization only — statement validation is unchanged.

## Generalization

The event-loop finding was originally scoped to `raw_query`, but the curated
tools had no bounds at all: `list_categories` returns every category, and
`spending_over_time` with `group_by_category` returns one row per month per
category, so a long-history file can produce a very large response. A shared
5000-row / 2 MB bound is now applied at the single registry dispatch point, so
both the MCP server and the CLI are covered and a tool added later cannot be
unbounded by omission. Oversized results are rejected rather than truncated —
these are financial aggregates, and a silently truncated total is a wrong
answer, not a smaller one.

## Testing

Validation tests previously sat inside live-database-gated blocks in both test
files, so on any machine without `QUICKEN_DB_PATH` the tests covering the
security boundary silently did not run. That gating had already hidden two
real defects. Validation needs no live data — it rejects before the database
is opened — so it moved to a synthetic-database suite that runs everywhere.

Raw_query coverage without a live database: **8 tests → 34**.
Full suite: 116 passed, and all 225 pass against a live Quicken database.

Two cleanup tests were mutation-checked: removing the SIGKILL fails one,
dropping the concurrency-slot release fails the other.

## Known remaining work (deliberately not in this PR)

Statement validation is still a regex blocklist, which rejects harmless text
such as `SELECT 'DROP'`, and there is no single-statement policy. Both are
tracked as follow-up work.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

---

## Step 2 — PR 2: `fix/sanitize-error-paths`

Diff vs upstream: 2 files, ~181 insertions.

```bash
gh pr create \
  --repo dweekly/quicken-mac-mcp \
  --base main \
  --head apisani1:fix/sanitize-error-paths \
  --title "fix(errors): redact known-sensitive paths by exact match, close punctuation leak" \
  --body-file fix-plans/pr-body-sanitize.md
```

### Body — save as `fix-plans/pr-body-sanitize.md`

```markdown
`sanitizeError()` strips filesystem paths from error messages so a tool
response cannot leak personal file locations. Its pattern had gaps, and
patching the pattern kept producing new ones: single-segment paths, then
spaced folder names, then punctuation.

The concrete leaks, all verified against unmodified `main`:

```
/Users/x/Documents/My Finances (2026).quicken/data → <path> Finances (2026).quicken/data
/Users/x/Documents/Antonio's Finances/data         → <path>'s Finances/data
/data                                              → not redacted at all
https://example.com/api                            → https:/<path>   (corrupted)
```

Rather than widen the pattern a fourth time, the two values we actually know
are sensitive — `QUICKEN_DB_PATH` (and its bundle directory) and the user's
home directory — are now redacted by exact string match first. Exact matching
cannot be defeated by spacing, punctuation, or Unicode in a folder name, so
the highest-value secret no longer depends on the pattern being exhaustive.

The pattern remains as a fallback for paths not known in advance. Punctuation
is allowed inside a segment but not at its end, so a sentence's final period
and a comma between two listed paths stay outside the redaction. Segment
chunks are length-bounded, because error text can echo caller-supplied SQL and
the nested quantifiers should not degrade into pathological backtracking.

One residue is documented in the code rather than papered over: the
quoted-path rules assume the quote character does not appear inside the path,
so a single-quoted path containing an apostrophe can still leave a fragment.
The exact-match pass covers the real database path in that case.

Tests cover the full matrix suggested in review — spaces, Unicode,
punctuation, apostrophes, ampersands, quoted and unquoted forms, URLs,
sentence-final periods, multiple paths in one message — plus exact-match
redaction of a path whose spacing would defeat the pattern on its own.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

---

## Step 3 — After opening

Add a comment to each PR recording the live-database validation from plans 1
and 2, since CI cannot reproduce it:

> Validated against a real Quicken for Mac database (not just the synthetic
> suite): full test suite green with `QUICKEN_DB_PATH` set, 0 skipped. Both
> branches were also tested merged together on a throwaway branch to cover the
> `formatToolError` path that only exists when both are present.

## Housekeeping

Once both PRs are open, delete the superseded branches — every one of their
commits is already an ancestor of `fix/raw-query-hardening`:

```bash
git push origin --delete fix/raw-query-limit-bypass
git push origin --delete fix/raw-query-pragma-bypass
git push origin --delete fix/raw-query-timeout
git branch -D fix/raw-query-limit-bypass fix/raw-query-pragma-bypass fix/raw-query-timeout
```
