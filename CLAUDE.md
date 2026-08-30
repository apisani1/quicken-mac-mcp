# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Read-only access to Quicken For Mac's Core Data SQLite database, shipped as two artifacts that share the same schema knowledge:

- **Skill** (`plugin/skills/quicken/SKILL.md` + `references/`) — teaches Claude to write SQL directly against the database. This is the recommended, primary path.
- **MCP server** (`src/`) — wraps 8 of those same SQL recipes as typed tools, for MCP clients that can't load skills.

The same binary (`qmac`, built from `src/index.ts`) is both the MCP server (stdio transport, default when run with no subcommand) and a self-documenting CLI that lets you run any tool directly from the terminal (`qmac list-accounts`, `qmac spending-by-category --help`, etc.).

**Quicken For Mac must be running** for any of this to work — Quicken encrypts its database file into an unreadable stub when the app is closed, and `src/db.ts` treats the absence of `ZACCOUNT` as the "still encrypted" signal.

## Build & Run Commands

- `npm run build` — compile TS to `dist/`
- `npm run dev -- [command] [args]` — run the CLI/server from source via `tsx` (no build step)
- `npm start` — run the compiled server (`dist/index.js`)
- `npm test` — full Vitest suite
- `npm run test:watch` — Vitest watch mode
- `npx vitest run src/__tests__/spending-tools.test.ts` — run a single test file
- `npx vitest run -t "some test name"` — run tests matching a name
- `npm run check:docs` — cross-checks the skill/docs Z-prefixed SQL identifiers and fenced SQL blocks against a synthetic (and optionally live) database; wraps `src/__tests__/docs.test.ts`
- `npm run lint` / `npm run format` / `npm run format:check`
- `npm run sync:versions` — propagate `package.json`'s version into `manifest.json`, `server.json`, `plugin/.claude-plugin/plugin.json`, and `Formula/qmac.rb` (also runs automatically on `npm version`)
- `npm run pack:mcpb` — build the `.mcpb` bundle for Claude Desktop drag-and-drop install

### Live vs. synthetic tests

Most tests run against an in-memory synthetic Quicken-shaped fixture (`src/__tests__/fixtures/quicken.ts`) — no real data, no Quicken required, deterministic edge cases (splits, transfers, report-excluded rows, missing `ZPOSTEDDATE`).

A second class of tests (`live-quicken.ts` fixture, used by `docs.test.ts` and others) opts into a **real** Quicken database:

- If `QUICKEN_DB_PATH` is unset, it auto-detects a single `.quicken` bundle in `~/Documents`; if none/multiple are found, live checks **skip with a warning**.
- If `QUICKEN_DB_PATH` **is** set, the suite treats it as a hard requirement — an unusable or invalid file **fails** the suite rather than skipping.

`npm test`'s `pretest` hook (`scripts/report-live-test-status.mjs`) prints up front whether live checks will run or skip, so a red/green run isn't ambiguous about coverage.

## Architecture

### Single source of truth: `src/tools/registry.ts`

Every MCP tool is defined once in `toolsRegistry` — name, description, a `ToolParamDef` per parameter (type, description, optional/required, Zod schema, enum values), and a `handler(db, args)`. Everything else is generated from this array:

- `src/server.ts` builds the MCP tool schemas straight from each `ToolParamDef.zod`.
- `src/index.ts` (the CLI) uses the same defs to parse `--flag` arguments (accepting snake_case/kebab-case/camelCase, coercing types, validating with the same Zod schema), and to generate `--help`, the general help screen, and `qmac man`.

Adding a tool means: write the handler in `src/tools/<name>.ts`, add one entry to `toolsRegistry`. CLI help, MCP schema, and argument parsing all follow automatically.

### `src/db.ts` — Core Data access layer

- Quicken stores dates as **Core Data timestamps** (seconds since 2001-01-01), not Unix time. Use `isoToCoreData` / `coreDataToIso` / `CORE_DATA_EPOCH_OFFSET` for conversions; `inclusiveEndDateToCoreDataExclusive` turns a user-facing inclusive end date into the correct half-open upper bound.
- Core Data entity IDs (`Z_ENT`) are **not stable across databases** — `getCategoryTagEntityId` looks up the real value from `Z_PRIMARYKEY` at runtime and caches it per-connection (`WeakMap`). Never hardcode a `Z_ENT` value.
- `createDbAccessor(dbPath)` returns a lazy, memoized getter: the server can start without a valid DB, opens it on first tool call, and re-opens automatically if Quicken swaps the underlying file (detected via inode change) — e.g. after the user quits and relaunches Quicken.
- DB path resolution order everywhere: explicit path argument → `QUICKEN_DB_PATH` env var → auto-detect the sole `.quicken` bundle in `~/Documents` (refuses to guess if there are 0 or 2+).
- All connections are opened `{ readonly: true }`. Never write to the decrypted database.

### `src/server.ts` — MCP wiring

- `sanitizeError` strips absolute filesystem paths from any error text before it reaches logs or tool responses — personal data file paths must never leak.
- `formatToolError` adds targeted troubleshooting text for known failure modes (native module version mismatch, database still encrypted, missing table) and, for the "encrypted" case, actually re-probes the DB via `isDatabaseDecrypted()` to give an accurate message.
- The `McpServer` constructor's `instructions` string is the tool-selection guidance an MCP client sees — keep it in sync with actual tool behavior/defaults (e.g. `spending_by_category`/`spending_over_time` defaulting to checking+creditcard accounts) when those change.

### Version propagation

`package.json`'s `version` is the single source of truth; `scripts/sync-versions.mjs` fans it out to `manifest.json`, `server.json`, and `plugin/.claude-plugin/plugin.json` (wired to the `npm version` lifecycle hook). `src/__tests__/versions.test.ts` fails the build if any of these — plus `package-lock.json` and `Formula/qmac.rb` — drift from `package.json`. Don't hand-edit version fields in those files; bump `package.json` and run `npm run sync:versions`.

### Docs/schema consistency

`docs/schema.md` is the full Core Data schema reference (84 entities). The skill's `SKILL.md` and `references/*.md` embed SQL against the same schema; `docs.test.ts` parses every Z-prefixed identifier out of those docs and verifies it resolves against a real table/column/index, and executes every fenced SQL block against the synthetic fixture (and live DB, if available). When you change a query pattern in the skill docs, run `npm run check:docs`.

## Code Guidelines

- **Style**: Prettier-enforced — double quotes, semicolons, 90-char print width, ES5 trailing commas (`.prettierrc`). `@typescript-eslint/no-explicit-any` is intentionally off.
- **ESM**: `"type": "module"`; all relative imports must include the `.js` extension (e.g. `import { x } from "./db.js"`), even though the source is `.ts`.
- **Database safety**: all Quicken access is read-only. Never issue writes, migrations, or `PRAGMA` changes against the decrypted database.
- **Error handling**: route errors through `sanitizeError`/`formatToolError` rather than surfacing raw messages — raw `Error.message` can contain absolute paths to the user's financial data file.
- **Tool parameter naming**: CLI accepts snake_case/kebab-case/camelCase; internal args are snake_case to match the Zod/`ToolParamDef` keys in `registry.ts`.
