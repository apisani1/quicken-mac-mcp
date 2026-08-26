# Plan 0 — Prepare "Secure Space" to run plans 1 and 2

Do this once, before plan 1. Everything here happens on the **Secure Space**
boot volume, where Quicken lives.

Plans 1 and 2 run there. **Plan 3 (opening the PRs) runs on the main volume**
— it needs no Quicken database, so nothing in this file is required for it.

---

## What you need, and what you already have

macOS ships two of these. The rest you may need to install.

| Tool | Needed for | Likely status |
|------|-----------|---------------|
| Node ≥ 22 | everything | install |
| npm | `npm ci`, `npm test` | comes with Node |
| git | clone, checkout, the plan-2 merge | usually present via Xcode CLT |
| `sqlite3` | preflight check that the database is decrypted | ships with macOS |
| `mdfind` | Spotlight search for your `.quicken` bundle | ships with macOS |
| Quicken for Mac | must be running with the database open | you have it |

`npx` and `tsx` need no separate install: `npx` ships with npm, and `tsx`
arrives as a dev dependency when you run `npm ci`.

`gh` is **not** needed here. It is only used in plan 3.

---

## Step 1 — Node

Install Node **24.x** if you have a choice of version. The reason is in step 4:
a matching Node major means the native SQLite module downloads a prebuilt
binary instead of compiling from source.

Any of these work:

```bash
# Option A — official installer
open https://nodejs.org/en/download    # take the macOS Apple Silicon .pkg

# Option B — Homebrew, if you already run it on this volume
brew install node@24

# Option C — nvm, if you want multiple versions
nvm install 24 && nvm use 24
```

Verify:

```bash
node -v      # v24.x preferred, v22+ required
npm -v
```

---

## Step 2 — git and the compile fallback

```bash
git --version
```

If git is missing, or if step 4 ends up needing to compile, install the Xcode
Command Line Tools:

```bash
xcode-select --install
python3 --version    # node-gyp needs python3; CLT usually provides it
```

This is insurance. On the happy path in step 4 nothing is compiled and none of
it is used — but discovering it is missing halfway through `npm ci` is worse
than installing it now.

---

## Step 3 — Clone the repository

The repo is **public**, so no SSH key, no GitHub login, no `gh`:

```bash
cd ~   # or wherever you keep code
git clone https://github.com/apisani1/quicken-mac-mcp.git
cd quicken-mac-mcp
git branch -r | grep -E 'hardening|sanitize'
```

You should see `origin/fix/raw-query-hardening` and
`origin/fix/sanitize-error-paths`. Those are the two branches the plans test.

You will also find the plans themselves in `fix-plans/` — the clone carries
them, so you do not need to copy anything across from the other volume.

---

## Step 4 — Install dependencies (and the native-module check)

```bash
npm ci
```

This is the step most likely to surprise you. `better-sqlite3` is a native
module; its install script is `prebuild-install || node-gyp rebuild`.

- **Prebuilt binary (fast, no compiler):** a matching prebuild exists for
  `better-sqlite3` 12.11.1 on Node 24 (ABI 137), darwin-arm64. If Secure Space
  runs the same Node major on the same Apple Silicon, this is what happens.
- **Compiled from source (slow, needs Xcode CLT + python3):** the fallback if
  the Node major differs, if Node is running under Rosetta, or if the prebuild
  download fails.

Either outcome is fine as long as it finishes. Confirm the module actually
loads:

```bash
node -e "const D=require('better-sqlite3'); const d=new D(':memory:'); console.log('better-sqlite3 OK', d.prepare('select sqlite_version() v').get());"
```

If this throws `NODE_MODULE_VERSION`, the module was built for a different
Node. Fix it with `rm -rf node_modules && npm ci` under the Node you intend to
use. **Never copy `node_modules/` from the other volume** — that is exactly
how this error happens.

Network access is required for `npm ci` regardless, and for the prebuild
download.

---

## Step 5 — Quicken and a copy of your database

Plan 1 has the full procedure for finding your `.quicken` bundle and working
from a copy. Two things to have settled before you start it:

1. **Quicken runs on this volume and can open your file.** The database is
   encrypted at rest — if Quicken is not running and holding the file open,
   every live test fails at the decryption check.
2. **You know roughly where your Quicken file lives**, or can find it via
   Quicken's own **File → Show in Finder**. It is *not* in `~/Documents`, so
   the tool's auto-detection will not find it and plan 1 will have you set
   `QUICKEN_DB_PATH` explicitly.

Plan 1 will have you copy the bundle and open the **copy** in Quicken, so your
real file is never the thing under test.

---

## Step 6 — Verify you are ready

```bash
node -v && npm -v && git --version && sqlite3 --version | head -1 \
  && node -e "require('better-sqlite3'); console.log('native module OK')" \
  && (pgrep -x Quicken >/dev/null && echo "Quicken running" || echo "Quicken NOT running — start it before plan 1")
```

All green? Go to **plan 1**.

---

## Getting results back to the main volume

Plans 1 and 2 write test output to `~/results-hardening.txt`,
`~/results-sanitize.txt`, and `~/results-combined.txt`. You will want those on
the other volume for review before plan 3. Options, easiest first:

- **A shared external volume.** `Travel Backup` was mounted on the main volume;
  if it mounts here too, copy the files there.
- **Mount the other volume.** From Secure Space, the main volume's data
  partition can be mounted in Finder or via `diskutil mount`, and the files
  copied directly.
- **Just paste them.** The final `Tests` line plus any failure output is
  usually all that matters, and it is short enough to paste into a session on
  the other side.

What to capture for each run:

- the final `Tests` line — passed / skipped / total
- any failing test names with their assertion output
- the output of the manual checks in plan 1 step 3–4 and plan 2 step 3
