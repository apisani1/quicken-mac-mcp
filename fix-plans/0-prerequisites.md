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

If you want to exceed 500 transactions without hand-entry, Quicken's
**File → Import** accepts QIF/CSV depending on account type; generating a file
of invented transactions is far faster than typing them. Verify what your
Quicken version accepts before investing time in generating one.

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
