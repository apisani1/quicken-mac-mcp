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

- **The copied Quicken bundle is still real financial data**, sitting in the
  same account the untrusted code runs in, and decrypted while Quicken is
  open. The account boundary does not protect the copy from code running
  inside that account. If you have a smaller or older Quicken file that would
  still exercise the schema, prefer it.
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

## Step 3 — Move a copy of the Quicken bundle into the test account

A standard user cannot read the administrator's protected folders, so the copy
has to be staged through a shared location. **Quit Quicken first**, so the
bundle is not mid-write.

**As the administrator:**

```bash
QUICKEN_SRC="/full/path/to/My Finances.quicken"      # <- your real bundle
sudo cp -R "$QUICKEN_SRC" /Users/Shared/quicken-test-copy.quicken
sudo chown -R quicken-test:staff /Users/Shared/quicken-test-copy.quicken
```

**Then, logged in as `quicken-test`:**

```bash
mkdir -p "$HOME/Documents"
mv /Users/Shared/quicken-test-copy.quicken "$HOME/Documents/quicken-test-copy.quicken"
chmod -R 700 "$HOME/Documents/quicken-test-copy.quicken"
ls -ld "$HOME/Documents/quicken-test-copy.quicken"     # expect drwx------
```

Three details that matter:

- `/Users/Shared` is world-writable (`drwxrwxrwt`). Do not leave the copy
  there — the `mv` above is the point, not a formality.
- The copy lives in the test account's `~/Documents`. That folder is
  TCC-protected, so the first time Terminal (or `node`) reads it macOS will
  prompt: *"Terminal would like to access files in your Documents folder."*
  **Approve it.** This is a narrow, per-app, per-account grant — it is not
  Full Disk Access, and it does not reach the administrator account.
- Quicken needs to **write** to the bundle it opens, which is why ownership is
  transferred rather than just read access.

One useful side effect: `~/Documents` is exactly where the tool's
auto-detection looks for a `.quicken` bundle, so it will find this copy on its
own. Set `QUICKEN_DB_PATH` explicitly anyway. With it set, an unusable
database makes `npm test` **fail** (exit 1); with auto-detection alone, the
live suites merely warn and skip — which is the silent-skip failure mode this
whole effort exists to eliminate.

Locating your original bundle has to happen from the account that owns it —
`mdfind` run as `quicken-test` will not see the administrator's files. Plan 1
step 0 has the search commands; run those as the administrator, then use the
path here.

---

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

Launch Quicken while logged in as `quicken-test` and open
`~/Documents/quicken-test-copy.quicken`.

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
export QUICKEN_DB_PATH="$HOME/Documents/quicken-test-copy.quicken/data"
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

The copy is real financial data. When testing is done:

```bash
rm -rf "$HOME/Documents/quicken-test-copy.quicken"
```

Keep the account itself if you expect a second round of testing; delete it
along with its home directory if not.
