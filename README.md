[![skills.sh](https://skills.sh/b/coderianx/golang-backend-style)](https://skills.sh/coderianx/golang-backend-style)

# golang-backend-style

An agent skill that teaches AI coding agents to write Go backend code
in a complete, opinionated idiom: a layered `internal/` architecture,
raw SQL over pgx (PostgreSQL) or modernc.org/sqlite (SQLite), strict
sentinel-error discipline, a fixed normalize/validate flow, a
distinctive vertical code format, and table-driven tests — distilled
from a production code base and generalized so any service can adopt
it.

## Install

The [skills CLI](https://github.com/vercel-labs/skills) (Node.js 22.20+)
installs the skill into your agent. Run it from your project root:

```sh
npx skills add coderianx/golang-backend-style
```

The CLI detects your agents, shows the skill it found, and asks where
to install it. Useful variants:

```sh
# Install for this project only, skipping prompts
npx skills add coderianx/golang-backend-style -y

# Install globally (available in every project on your machine)
npx skills add coderianx/golang-backend-style -g

# Install for specific agents (e.g. OpenCode and Claude Code)
npx skills add coderianx/golang-backend-style -a opencode -a claude-code

# Preview what the repository contains, without installing
npx skills add coderianx/golang-backend-style --list

# Copy files instead of symlinking, if your agent ignores symlinks
npx skills add coderianx/golang-backend-style --copy
```

Project installs write a `skills-lock.json` recording the source and
content hash of the skill — commit it so your team shares the exact
same version.

Keep the skill current:

```sh
npx skills update
```

### Manual install

Without the CLI, copy `SKILL.md` into your agent's skills folder as
`golang-backend-style/SKILL.md`:

| Agent      | Project folder       | Global folder            |
| ---------- | -------------------- | ------------------------ |
| Claude Code | `.claude/skills/`   | `~/.claude/skills/`      |
| OpenCode   | `.opencode/skills/` | `~/.config/opencode/skills/` |

## What it covers

- **Stack**: chi routing, raw SQL through a global pool or handle —
  pgx/v5 (PostgreSQL) or modernc.org/sqlite (SQLite) — JWT (HS256) +
  bcrypt auth, godotenv config, stdlib testing.
- **Architecture**: one-way layering `handlers → repository → store`;
  per-domain handler packages with a `handlers.go` / `helpers.go`
  split; models as pure data.
- **Handlers**: a fixed step order, `{"error": ...}` response bodies
  built with `map[string]any`, `http.Status*` constants, and security
  that never leaks which step failed.
- **Repository**: raw multi-line SQL with `$n` placeholders, sentinel
  errors declared above the function that returns them, `EXISTS`-based
  access checks resolved in a single query, transactions with
  `defer tx.Rollback`.
- **Validation**: normalize/validate separation, password-like fields
  never trimmed or normalized, presence-only checks on lookup flows.
- **Schema**: idempotent `CREATE TABLE IF NOT EXISTS` bootstrap,
  partial unique indexes, nullable-column backfills for older
  databases.
- **Tests**: named table cases with got/want messages, DB-backed tests
  behind an `integration` build tag that skip — never fail — without a
  database.

## When it activates

The skill advertises itself through its `description` frontmatter, so
agents load it whenever they write or edit Go backend code in a project
that follows (or should follow) this architecture — new endpoints,
handlers, repository functions, models, middlewares, schema, or tests.
In OpenCode it can also be loaded explicitly with
`@golang-backend-style`.

Nothing runs at install time: a skill is instructions, not code. As
with any third-party skill, review [`SKILL.md`](./SKILL.md) before
installing it.

## Compatibility

Works with every agent the skills CLI supports — Claude Code,
OpenCode, Codex, Cursor, Gemini CLI, GitHub Copilot, and dozens more.
The `SKILL.md` format (YAML frontmatter + Markdown) is the
cross-agent standard, so the same file also works when installed
manually.

## License

[MIT](./LICENSE) © coderianx
