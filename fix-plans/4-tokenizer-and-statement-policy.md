# Plan 4 — Replace the regex blocklist with a real statement policy

Independent follow-up work. Not a blocker for plans 1–3; it becomes its own
PR after those land.

This addresses the two findings deliberately deferred during the security
review:

1. Statement validation is a regex blocklist, which is both bypassable and
   wrong in the other direction — it rejects harmless queries.
2. There is no single-read-statement policy with explicit comment and
   semicolon handling.

---

## Current state (read this before starting)

One item from the original write-up is **already fixed** and should not be
re-done: `inner.replace(/;\s*$/, "")` is gone. Trailing terminators are now
handled by `stripTrailingTerminator()` in `src/tools/raw-query.ts`, which
walks the string tracking string literals, quoted identifiers, and line/block
comments. `SELECT ';' as s` survives it intact.

What genuinely remains is the policy itself, in `src/tools/raw-query.ts`:

```ts
if (!/^SELECT\s/i.test(trimmed)) throw new Error("Only SELECT queries are allowed");

const blocked =
  /\b(INSERT|UPDATE|DELETE|DROP|ALTER|CREATE|REPLACE|ATTACH|DETACH|PRAGMA|PRAGMA_\w*)\b/i;
if (blocked.test(trimmed)) throw new Error("Query contains disallowed statements");
```

Two problems:

- **False positives.** `SELECT 'DROP' as label` is rejected. So is
  `SELECT * FROM t WHERE note LIKE '%delete%'` — a completely ordinary query
  over financial memo text, which is exactly the kind of thing this tool is
  for.
- **Blocklists enumerate badness.** Every new SQLite keyword or function is a
  potential gap, and the `pragma_*()` finding was one such gap already.

---

## The key insight: SQLite can answer this itself

`better-sqlite3` exposes the parser. Preparing a statement compiles it without
executing it, and the compiled statement reports whether it only reads.

Measured behavior (run against `better-sqlite3` in this repo, on a read-only
connection):

| SQL | `prepare()` | `stmt.readonly` |
|-----|-------------|-----------------|
| `SELECT 'DROP' as s` | ok | **true** |
| `SELECT * FROM t WHERE note LIKE '%delete%'` | ok | **true** |
| `WITH c AS (SELECT 1 AS v) SELECT * FROM c` | ok | **true** |
| `SELECT 1 -- trailing comment` | ok | **true** |
| `SELECT 1;` | ok | **true** |
| `DELETE FROM t` | ok | **false** |
| `INSERT INTO t VALUES (2,'y')` | ok | **false** |
| `CREATE TABLE x (a INTEGER)` | ok | **false** |
| `SELECT 1; SELECT 2` | **throws** "contains more than one statement" | — |
| `PRAGMA table_info(t)` | ok | true ⚠️ |
| `SELECT * FROM pragma_table_info('t')` | ok | true ⚠️ |
| `SELECT * FROM pragma_database_list()` | ok | true ⚠️ |
| `ATTACH DATABASE '/tmp/evil.db' AS evil` | ok | true ⚠️ |

Two conclusions:

- **The single-statement policy is free.** `prepare()` rejects multiple
  statements natively, with correct comment and literal handling. Finding 2 is
  solved by using it.
- **`readonly` alone is not sufficient.** `ATTACH` and every `PRAGMA` form
  report `readonly === true`. `ATTACH` is the serious one: it can open another
  database file. These need an explicit deny that survives the rewrite.

---

## Proposed design

Replace the two regex checks with:

1. **Prepare the caller's SQL** on the read-only connection. A failure is a
   syntax error or a multi-statement attempt; report it as such.
2. **Require `stmt.readonly === true`.** This covers INSERT/UPDATE/DELETE/
   DROP/ALTER/CREATE/REPLACE and anything else that writes, without naming any
   of them.
3. **Keep a narrow, explicit deny** for the cases SQLite considers read-only
   but we do not want: `ATTACH`, `DETACH`, and PRAGMA in both spellings (bare
   `PRAGMA x` and the `pragma_*()` table-valued functions). This list is short,
   justified, and each entry has a stated reason — unlike the current
   catch-all.
4. **Keep the read-only connection** as the last line of defense. It is what
   makes a mistake in steps 1–3 non-fatal.

Note that the deny in step 3 can now be applied to the *parsed* statement
rather than the raw text, so `SELECT 'PRAGMA' as label` is no longer a false
positive.

### Where validation runs

`rawQuery` validates in the parent process and executes in a child. Preparing
in the parent is fine — `prepare()` parses without executing, so it does not
reintroduce the event-loop blocking the child process exists to prevent.
Validate the caller's inner SQL before wrapping, so error messages point at
the user's query rather than the generated wrapper.

---

## Implementation steps

1. Add `validateStatement(db, sql)` to `src/tools/raw-query.ts`, returning
   either the prepared statement or throwing a clear error.
2. Delete the `^SELECT\s` test and the `blocked` regex.
3. Apply the narrow deny list to the prepared statement's source.
4. Keep `stripTrailingTerminator()` unchanged — still needed for the subquery
   wrapper, and unrelated to policy.
5. Update the `raw_query` tool description in `src/tools/registry.ts`, which
   currently tells callers only SELECT statements are allowed.

## Tests to add (synthetic suite, never live-gated)

- **Regressions that must keep passing:** every current rejection —
  INSERT/UPDATE/DELETE/DROP/ALTER/CREATE, `ATTACH`, bare `PRAGMA`, the
  `pragma_*()` functions, multi-statement, empty and whitespace-only input.
- **False positives that must now pass:** `SELECT 'DROP' as label`,
  `SELECT * FROM t WHERE note LIKE '%delete%'`, `SELECT 'PRAGMA' as label`,
  a CTE, and a column literally named `delete_flag`.
- **Statement policy:** `SELECT 1; SELECT 2` rejected with a message that says
  one statement only, not a generic syntax error.
- **Error quality:** a genuine syntax error still produces a message that
  names the problem.

## Risks and how to handle them

- **Behavior change for callers.** Queries that were rejected will now
  succeed. That is the point, but it belongs in the changelog.
- **`prepare()` touches the database.** It needs a valid connection, so
  validation now fails differently when the database is unreadable. Make sure
  that error is distinguishable from "your SQL is invalid".
- **`readonly` semantics are SQLite's, not ours.** The ⚠️ rows in the table
  above are the known divergences. If a future SQLite version adds another
  read-only-but-unwanted construct, the deny list is where it goes — and the
  read-only connection still holds underneath.

## Definition of done

- No regex blocklist remains in `raw-query.ts`.
- Every rejection test from the current suite still passes.
- The false-positive queries above succeed.
- `npm test` green, including a live-database run per plan 1.
