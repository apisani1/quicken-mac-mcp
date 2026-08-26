# Plan 0 — Prepare a dedicated `quicken-test` account on "Secure Space"

Do this once, before plan 1. Everything happens on the **Secure Space** boot
volume.

Plans 1 and 2 run from a dedicated **standard (non-admin)** account named
`quicken-test`, so that `npm` install scripts, Node, and the test suite never
run with administrator privileges or with access to the administrator
account's data.

Plan 3 (opening the PRs) is **not** done here. It needs no Quicken database,
and it does need GitHub credentials — which is exactly what should stay out of
this account. Run it from your normal working volume once plans 1 and 2 are
green.

---

## Threat model — what this does and does not buy

**The risk being mitigated:** `npm ci` executes install scripts. This repo's
`better-sqlite3` dependency has one (`prebuild-install || node-gyp rebuild`),
so arbitrary package code does run during setup. A compromised dependency
anywhere in the tree runs with the privileges of whoever typed `npm ci`.

**What the separate account protects:** a compromised process is confined to
what `quicken-test` can reach. It cannot read the administrator account's
protected data, cannot install software system-wide, and cannot escalate
without admin credentials it does not have.

**What it does not protect — read this part:**

- **Whatever database you point the tests at is readable by code running in
  that account**, and is decrypted while Quicken is open. The account boundary
  does not protect it from code running inside the account. This is why step 3
  builds a **synthetic** Quicken file: with no real financial data in the
  account, there is nothing meaningful left to exfiltrate, and the isolation
  and the data protection stop depending on each other.
- **Network egress is unrestricted.** Anything readable in the account can be
  exfiltrated.
- **Home directories are group-readable at the top level.** macOS creates them
  `drwxr-x---` with group `staff`, and every local user is in `staff`. The
  administrator's `~/Documents`, `~/Desktop` and `~/Downloads` are `0700` and
  additionally TCC-protected, but do not assume the whole home is unreadable.
  Step 3 sets the copy to `0700` for the same reason.
- **It does not protect against compromise of a Secure Space administrator
  account** — as you noted.

---

## Step 1 — Create the account (administrator)

System Settings → Users & Groups → Add User:

- **Account type: Standard.** Not "Administrator", not "Sharing Only".
- Name it `quicken-test`.
- **Never** grant it administrator rights, not even temporarily.

---

## Step 2 — System-wide prerequisites (administrator, only if missing)

Check from the admin account what already exists system-wide, since anything
installed there is available to `quicken-test` too:

```bash
git --version        # provided by Xcode Command Line Tools
node -v              # needs >= 22; 24.x preferred (see step 4)
sqlite3 --version    # ships with macOS
```

Install only what is missing:

```bash
xcode-select --install     # git, and the compiler fallback for node-gyp
```

**Node — prefer the no-admin route.** Installing the official `.pkg` requires
admin and puts Node system-wide. Installing `nvm` inside the `quicken-test`
account requires no admin at all and keeps the toolchain inside the boundary:

```bash
# run this later, logged in AS quicken-test — not as admin
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
exec "$SHELL" -l
nvm install 24 && nvm use 24
```

Use the `.pkg` only if you would rather not run an install script from a URL.
Either way: **after installing, use Node and git only from `quicken-test`**.

---

## Step 3 — Build a synthetic Quicken file

Create a **new Quicken file with synthetic data** rather than copying your
real one. Quicken itself creates the file, so the schema is genuine — which is
the whole point of a live run — while the contents are invented.

Do this while logged in as `quicken-test`: **File → New**, saved to
`~/Documents/`.

### Name it to earn extra test coverage

```
~/Documents/My Test Finances (2026).quicken
```

The spaces and parentheses are deliberate. Plan 1 and plan 2 check that error
messages do not leak filesystem paths, and punctuated, spaced folder names are
exactly the case that broke the old sanitizer. A boring name tests less.

Then, in the `quicken-test` session:

```bash
export QUICKEN_DB_PATH="$HOME/Documents/My Test Finances (2026).quicken/data"
chmod -R 700 "$HOME/Documents/My Test Finances (2026).quicken"
```

Quote the path everywhere — it contains spaces.

### Hard requirements the live tests impose

Two of these are not negotiable, and both constrain which accounts and years
you export. They come from values hard-coded in the live suites:

1. **Calendar year 2024 must be covered.** `2024-01-01`, `2024-12-31` and
   `2024-06-30` appear 22 times across the live suites, and several of those
   assertions require non-empty results. A sample of "a limited number of
   years" that omits 2024 will fail tests that have nothing to do with the
   code. Exporting 2023–2025 gives 2024 plus range boundaries on either side,
   which the narrow-date-range and newest-first-ordering tests want.

2. **An account with `ZTYPENAME` of exactly `CHECKING`, holding
   transactions.** `list_accounts` filters on it and asserts a non-empty
   result; `query_transactions` filters `account_types: ["checking"]` and does
   the same.

3. **A credit card account with transactions dated in 2024.**
   `spending_over_time` is asserted with `account_types: ["creditcard"]` over
   `2024-01-01`–`2024-12-31` and expects rows back.

Your plan to sample **banking, credit cards, cash, assets and brokerage** is
more than these need, and the extra types are genuinely useful — they exercise
account-type filtering, the sorted-by-name listing, and the cross-tool check
that every transaction's account name appears in `list_accounts`.

One thing to keep in mind while choosing volumes: the spending tools default
to `checking` and `creditcard` only. Cash and asset accounts broaden account
coverage but contribute nothing to the spending suites, so put the bulk of
your transaction volume in checking and credit card accounts.

Brokerage is the one type CSV alone will not deliver — see below.

### What the file has to contain

This is the part that decides whether plan 1 is meaningful. **46 assertions
across 18 live suites require non-empty results**, and a sparse file makes
them fail in ways that look like code bugs but are only missing data. Build
for this list:

| Data | Why | Suites that fail without it |
|------|-----|------------------------------|
| ≥1 checking **and** ≥1 credit card account | the spending tools default to those two types | `list_accounts`, both spending suites |
| Categories, expense **and** income, with parent/child pairs | the hierarchy is asserted directly | `list_categories`, `getCategoryTagEntityId`, `category hierarchy integrity` |
| A few hundred transactions with payees, spread over ≥2 years | date bucketing, sorting, monthly aggregation | `query_transactions`, `date handling`, `spending_over_time` |
| Negative-amount categorized spending | every spending total | `spending_by_category`, `spending aggregation integrity` |
| Distinct payee names | substring search | `search_payees`, `payee search` |
| ≥1 brokerage account with securities, holdings and quotes | portfolio joins across ZLOT/ZPOSITION/ZSECURITY | `list_portfolio` (5), `portfolio data integrity` (2) |
| Some transfers, some splits, some uncategorized, some excluded-from-reports | these are the exclusions the spending tools apply | `cross-tool consistency`, `edge cases` |

The brokerage account is the fiddliest — add the account, buy a couple of
securities, and let Quicken download or hand-enter quotes. Skipping it costs
you 7 assertions.

### Recommended way to populate it: scrubbed CSV from your real file

Hand-entering hundreds of transactions is miserable, and a hand-built file
tends to be too tidy to be a useful test. Exporting a subset of your real data,
scrubbing anything identifying, and importing that into the new file gives you
realistic shapes — real payee distributions, real category spread, real date
gaps — with the identity removed.

**Do the export and the scrub from the administrator account** that owns the
real file. Only the scrubbed CSV crosses into `quicken-test`; the real bundle
never does.

1. In Quicken, with your real file open: export the registers you want
   (a couple of accounts, a couple of years) to CSV.
2. Scrub the CSV before it leaves the admin account. What identifies you:

   | Field | Treatment |
   |-------|-----------|
   | Account name | rename — "Checking", "Card", "Brokerage" |
   | Payee | replace with invented names; keep the *distribution* (a few frequent, many rare) so payee search is still meaningful |
   | Memo / notes | drop entirely — free text is where surprises hide |
   | Check numbers | drop |
   | Amounts | jitter by a few percent; salary and rent amounts identify you on their own |
   | Dates | keep — the date spread is what makes the monthly bucketing tests real |

   Keep categories as they are unless your category names are personal.
3. Move only the scrubbed CSV to `/Users/Shared`, import it into the new file
   from the `quicken-test` session, then delete it from `/Users/Shared`.

### What CSV import will not reproduce

Worth knowing before you assume the file is complete. These are schema
features the tools depend on, which a register CSV does not carry:

- **Transfers.** The spending tools exclude them via `ZTARGETACCOUNT`,
  `ZSENDACCOUNT` and `ZTRANSFER`. An imported CSV row usually becomes an
  ordinary categorized transaction, not a linked transfer, so that exclusion
  path goes untested. Create a few real transfers by hand between two imported
  accounts.
- **Splits.** One CSV row generally imports as one
  `ZCASHFLOWTRANSACTIONENTRY`. The split handling — including the
  negative-split convention the spending tools rely on — needs a handful of
  hand-entered split transactions.
- **Excluded-from-reports.** `ZEXCLUDEFROMREPORTS` is not a CSV column. Flag a
  couple of transactions manually in Quicken.
- **Investments.** Holdings come from `ZLOT`/`ZPOSITION`/`ZSECURITY` and prices
  from `ZSECURITYQUOTE`. A banking CSV creates none of them. To cover
  `list_portfolio` at all, add a brokerage account and enter a couple of buys
  so Quicken builds the lots.

One upside: CSV-imported transactions often have a null `ZPOSTEDDATE`, and the
code has a `COALESCE(ZPOSTEDDATE, ZENTEREDDATE)` fallback specifically for
that case. `integration.test.ts` has a test that only does real work when such
rows exist, so a CSV-built file may exercise that path better than a real one.

### Then add by hand what the import could not carry

Roughly 15–20 records, once. Everything above is bulk; this is the part that
makes the file structurally complete:

- [ ] **3–4 transfers** between two of the imported accounts (e.g. checking →
      credit card payment). Enter them as real transfers in Quicken, not as
      categorized transactions, so `ZTARGETACCOUNT`/`ZSENDACCOUNT` get set.
- [ ] **4–5 split transactions**, at least one with both a positive and a
      negative line, since the spending tools rely on the negative-split
      convention.
- [ ] **2–3 transactions flagged "exclude from reports"**, which sets
      `ZEXCLUDEFROMREPORTS`.
- [ ] **3–5 uncategorized transactions**, so the `(Uncategorized)` bucket both
      spending tools emit is non-empty.
- [ ] **1 brokerage account** with 2 securities, a couple of buy transactions
      each so Quicken builds the lots, and quotes (downloaded or hand-entered).
      This is the fiddliest item and the one worth 7 assertions.
- [ ] **At least one closed account**, so the active/closed flags in
      `list_accounts` are not uniformly identical.

Do these *after* the CSV import — importing into an account you have already
hand-edited is more error-prone than the reverse.

### Check what you actually ended up with

After importing, measure the file against what the suites need — this takes
seconds and saves you triaging phantom failures later:

```bash
Q="$HOME/Documents/My Test Finances (2026).quicken/data"
sqlite3 "$Q" "SELECT ZTYPENAME, COUNT(*) FROM ZACCOUNT GROUP BY ZTYPENAME;"
sqlite3 "$Q" "SELECT COUNT(*) AS transactions FROM ZTRANSACTION;"
sqlite3 "$Q" "SELECT COUNT(*) AS payees FROM ZUSERPAYEE;"
sqlite3 "$Q" "SELECT COUNT(*) AS categories FROM ZTAG;"
sqlite3 "$Q" "SELECT COUNT(*) AS transfers FROM ZTRANSACTION WHERE ZTARGETACCOUNT IS NOT NULL OR ZSENDACCOUNT IS NOT NULL;"
sqlite3 "$Q" "SELECT COUNT(*) AS splits FROM (SELECT ZPARENT FROM ZCASHFLOWTRANSACTIONENTRY GROUP BY ZPARENT HAVING COUNT(*) > 1);"
sqlite3 "$Q" "SELECT COUNT(*) AS excluded FROM ZTRANSACTION WHERE COALESCE(ZEXCLUDEFROMREPORTS,0) = 1;"
sqlite3 "$Q" "SELECT COUNT(*) AS lots FROM ZLOT;"
sqlite3 "$Q" "SELECT COUNT(*) AS quotes FROM ZSECURITYQUOTE;"
sqlite3 "$Q" "SELECT COUNT(*) AS null_posted FROM ZTRANSACTION WHERE ZPOSTEDDATE IS NULL AND ZENTEREDDATE IS NOT NULL;"

# The three hard requirements, verified rather than assumed:
sqlite3 "$Q" "SELECT COUNT(*) AS checking_accounts FROM ZACCOUNT WHERE UPPER(ZTYPENAME)='CHECKING';"
sqlite3 "$Q" "
  SELECT UPPER(a.ZTYPENAME) AS type, COUNT(*) AS txns_in_2024
  FROM ZTRANSACTION t JOIN ZACCOUNT a ON t.ZACCOUNT = a.Z_PK
  WHERE strftime('%Y', COALESCE(t.ZPOSTEDDATE, t.ZENTEREDDATE) + 978307200, 'unixepoch') = '2024'
  GROUP BY UPPER(a.ZTYPENAME);"
```

Required minimums, all three of which the live suites assert directly:

| Check | Must be |
|-------|---------|
| `checking_accounts` | ≥ 1 |
| `txns_in_2024` for `CHECKING` | ≥ 1 |
| `txns_in_2024` for `CREDITCARD` | ≥ 1 |

Every count that comes back zero tells you which suites will fail for want of
data rather than for want of correct code. `transfers`, `splits`, `excluded`
and `lots` are the four most likely zeros after a CSV-only import.

### What a synthetic file cannot tell you

Real data volume. Plan 1 asks whether the new 5000-row shared cap is too low
for a long-history file, and a synthetic file cannot answer that. Plan 1 step 3
has a safe way to check it against your real file without running any test
code against it.

## Step 4 — Log in as `quicken-test` and set up the repo

Everything from here on is in the `quicken-test` session.

```bash
cd ~
git clone https://github.com/apisani1/quicken-mac-mcp.git
cd quicken-mac-mcp
npm ci
```

The repo is public, so the clone needs no SSH key and no GitHub login. Do not
copy the administrator's `~/.ssh` or git credentials into this account.

`npm ci` is the step that runs untrusted install scripts. Add
`--foreground-scripts` if you want to see exactly what they do:

```bash
npm ci --foreground-scripts
```

`--ignore-scripts` is not an option here: `better-sqlite3` would never get
built and every test would fail.

Confirm the native module loads:

```bash
node -e "const D=require('better-sqlite3'); const d=new D(':memory:'); console.log('OK', d.prepare('select sqlite_version() v').get());"
```

A prebuilt binary exists for `better-sqlite3` 12.11.1 on Node 24 (ABI 137),
darwin-arm64 — with a matching Node major, nothing is compiled. A different
Node major, Rosetta, or a failed download falls back to compiling from source,
which needs the Xcode CLT from step 2. If you see a `NODE_MODULE_VERSION`
error: `rm -rf node_modules && npm ci` under the Node you intend to use, and
never copy `node_modules/` between accounts or volumes.

---

## Step 5 — Quicken, in the test account

Launch Quicken while logged in as `quicken-test` and open your synthetic file,
`~/Documents/My Test Finances (2026).quicken`.

- You will likely have to **sign in with your Quicken ID** in this account,
  since subscription state is per-user. That places Quicken credentials inside
  the test account — a smaller exposure than the financial data already there,
  but worth knowing before you type them.
- **Leave Quicken running** for the whole session. It decrypts the database
  only while open; closing it re-encrypts and every live suite fails.
- **Do not grant Full Disk Access** to Terminal, Node, or anything else.
  Quicken opening a file inside its own home does not need it.

---

## Step 6 — Verify you are ready

```bash
node -v && npm -v && git --version && sqlite3 --version | head -1 \
  && node -e "require('better-sqlite3'); console.log('native module OK')" \
  && (pgrep -x Quicken >/dev/null && echo "Quicken running" || echo "Quicken NOT running") \
  && id -Gn | tr ' ' '\n' | grep -qx admin && echo "WARNING: this account has admin rights" \
  || echo "account is non-admin (good)"
```

Then confirm the database copy is actually decrypted:

```bash
export QUICKEN_DB_PATH="$HOME/Documents/My Test Finances (2026).quicken/data"
sqlite3 "$QUICKEN_DB_PATH" ".tables" | tr ' ' '\n' | grep -c ZACCOUNT   # expect >= 1
node scripts/report-live-test-status.mjs; echo "exit=$?"                # expect exit=0
```

Ready? Go to **plan 1**.

---

## Step 7 — Getting results out

Transfer **only the text results** — never the database copy.

Plans 1 and 2 write to `~/results-hardening.txt`, `~/results-sanitize.txt` and
`~/results-combined.txt` inside the test account. To move them:

```bash
cp ~/results-*.txt /Users/Shared/          # readable from the admin account
```

and delete them from `/Users/Shared` once collected. What actually matters is
short enough to paste by hand: the final `Tests` line, any failure output, and
the results of the manual checks in plan 1 and plan 2.

Those results are the input to plan 3, which you run back on your normal
working volume.

## When you are finished

The synthetic file holds no real data, so there is nothing urgent to destroy —
keep it for the next round of testing rather than rebuilding it. Building a
usable one is the most tedious part of this whole setup, and plan 4 will want
a live database again.

Keep the `quicken-test` account for the same reason.
