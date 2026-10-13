# Siding — local agent handoff

> **Local-only context.** Gitignored. Read this first if you are an isolated agent picking up this repo.
>
> Repo: `c:\source\siding` · Module: `github.com/TheBlackhowling/siding` · License: MIT

---

## What we are building

**Siding** is a stack-agnostic named local-dev CLI extracted from the ideas in PosterChild PR 831 / `pcaicode-betty` (`docker/PCAI_CLI.md`, `resolve-stack.mjs`).

**Metaphor:** railroad siding — a parallel track where a checkout parks its Docker stack (ports, compose project, `*.localhost` hosts) without copying allocator scripts into every app.

**Not:** vinyl siding. **Not:** an npm wrapper. **Not:** a CLI-in-Docker for daily use.

**Distribution plan:** Go binary via GoReleaser (Windows/macOS/Linux), Scoop/Homebrew/install scripts. No npx.

**Prior art:** `c:\source\pcaicode-betty` — host-side `docker compose`, `*.localhost`, Caddy `http_port` 1355 (not host :80), lock path normalization, shared Caddy proxy.

---

## Locked-in architecture

### Per repo (checked in)

| File | Role |
|------|------|
| `.siding/config.yml` | Recipe: project, modes, compose files, port keys, host/compose patterns |
| `.siding/compose/` | Optional committed copies of compose (adapted). Originals at repo root stay for non-siding users |
| `.siding/.gitignore` | Ignores generated `env` |
| `.siding/env` | Generated slot env (not committed) |

### Machine state (user data dir — **not** beside exe, **not** in siding source repo)

```text
Windows:  %LOCALAPPDATA%\siding\state.json
macOS:    ~/Library/Application Support/siding/state.json
Linux:    ~/.local/share/siding/state.json

  checkouts:  { normalized_repo_root_path → { project, stack, modes: { ports... } } }
  locks:      { project:stack:mode → { checkout, compose_project } }
  siding-proxy/   (Caddyfile + compose; siding-proxy-caddy on 127.0.0.1:1355)
```

**Overrides:** `SIDING_DATA` (data root), `SIDING_ROOT` / `--root` (state file or dir).

### Resolution chain

1. `startPath` = `--path` or cwd
2. Walk up to `.siding/config.yml` (legacy `.siding.yml` still accepted) → **repo root**
3. **Checkout key** = `NormalizeLockPath(repo_root)` (Windows-safe compare)
4. **Stack** = `--stack` → folder basename minus `folder_prefix` → basename
5. Host / compose project = expanded `{stack}.{project}.{mode}` patterns (duplicate segments collapsed)
6. Ports = `state.checkouts[key].modes[mode].ports` if present, else mode defaults

### Deliberately removed / rejected

- Workspace slug dirs (`.siding-workspace`, `SIDING_WORKSPACE`, `--workspace`)
- Overlay JSON beside checkout parent (`pcai-stacks.json` pattern moved into path-keyed `state.json`)
- **`main` / `default_stack`** — folder is the stack
- **Named stack roster in the contract** (`stacks.betty`, `match:`) — repo defines *kinds* (modes), not slot names

### Stack naming (no `main`, no roster)

```yaml
project: pcai
folder_prefix: pcaicode-
host: "{stack}.{project}.{mode}"
compose_project: "{project}-{stack}-{mode}"
modes:
  test: { compose_files: [docker-compose.yml], ports: { web: 3000, api: 8000 } }
```

`pcaicode-betty` → stack `betty`. New folder `pcaicode-ruth` needs no contract edit.

Resolution: `--stack` → strip `folder_prefix` → basename.

---

## CLI UX (locked in)

```text
siding init [--path PATH] [--project NAME] [--compose FILE] [--port KEY=N] [--folder-prefix P] [--force]
siding sync [--path PATH] [--stack NAME] [--mode test|prod] [--print-hosts] [--json] [--dry-run]
siding plan [--path PATH] [--stack NAME] [--mode test|prod] [--json]
siding doctor [--path PATH]
siding add [--path PATH] [--stack NAME] [--mode test|prod] [--json] [--dry-run]
siding up|down|ps [--path PATH] [--stack NAME] [--mode test|prod] [--profile NAME] [--build]
siding proxy up [--listen PORT] [--json]
siding proxy down|reload|ps
```

Run from repo or pass `--path`; walks up to `.siding/`. No `package.json` wrapper required.

---

## Implementation status

### Done

- `siding version`, `siding init`, `siding sync`, `siding doctor`, `siding plan`
- `siding add` — globally unique port block into `state.checkouts[path]` (stride 10; idempotent)
- `siding up` / `down` / `ps` — write `.siding/env`, claim/release lock, host-side `docker compose`
- `siding proxy` — shared Caddy on `127.0.0.1:1355` (`http_port` rewritten off :80); admin API on `:2019` for `proxy reload`; TCP keys (redis) are Docker binds only
- Contract is modes-only under committed `.siding/config.yml` (legacy `.siding.yml` still loads)
- `siding sync` maps committed `.siding/config.yml` onto `state.json` + `.siding/env`; `init` on an existing recipe runs sync instead of erroring
- `siding init` / `doctor` / `plan` scan compose for hardcoded host ports, NEXT_PUBLIC_/VITE_/DEV_* :port defaults, isolated networks, host network, localhost internals (skips healthchecks / GCS)
- Packages: `contract`, `resolver`, `state`, `paths`, `doctor`, `compose`, `proxy`, `advise`, `cli`
- Path-keyed `state.json` read/write, lock helpers, port collection + allocation
- Example fixture: `examples/hello/.siding/config.yml`
- CI (Go test matrix), GoReleaser scaffold
- **Docker dev without local Go:** `docker-compose.dev.yml`, `docker/build.ps1` → `bin/siding.exe`
- **Independent compose files** — `docker-compose.prod-db.yml` is its own mode (ports, `-p` project, `.siding/env.<mode>`), not merged into `test`. Overrides stay with the base.

### Not done (next work)

1. PosterChild adapter — `.siding/config.yml` for pcaicode + migrate off in-repo scripts
2. First release — tag `v0.1.0`, GoReleaser, install scripts
3. `siding doctor --fix-path` — migrate state when repo moves on disk (discussed, not built)

### Known packages deleted

- `internal/overlay/` — superseded by `state.json`
- `internal/lock/` — locks live in `state.json`

---

## Dev workflow (no local Go required)

Tim runs on Windows without Go installed. Use Docker to test/build:

```powershell
cd c:\source\siding

# Tests
docker compose -f docker-compose.dev.yml run --rm dev sh -c "go mod tidy && go test ./..."
# or
.\docker\dev.ps1 test

# Build Windows exe on host (cross-compile in container, output via volume mount)
.\docker\build.ps1
.\bin\siding.exe plan --path examples\hello --json
.\bin\siding.exe add --path examples\hello --root .docker\siding-data --dry-run
.\bin\siding.exe up --path examples\hello
.\docker\dev.ps1 doctor --path examples\hello
```

`up` / `down` / `ps` must be the host exe (`.\docker\build.ps1` then `.\bin\siding.exe`). Do not `dev.ps1 run up` — compose cwd would be `/src/...`.

**Volumes:** repo → `/src`; machine state dev copy → `.docker/siding-data` (`SIDING_DATA=/data` in container).

**Important:** checkout keys differ by runtime path:
- Native Windows exe: `c:/source/siding/examples/hello`
- Docker `go run`: `/src/examples/hello`

Use one runtime consistently per checkout, or migrate state entries.

---

## User environment context

### Scale target (for the record, Sep 2026)

Tim plans to run **~6 PosterChild feature slots in parallel** (separate branches/worktrees/clones, each with its own Docker stack), **plus** active parallel work in **other repos** (siding, codencil, etc.) with live testing as he goes.

Implications for siding and dev layout:

- **6 pcai stacks** → one committed recipe + 6 `state.json` checkouts (ports globally unique). Folder `pcaicode-ruth` is stack `ruth` via `folder_prefix` — never a shared `main` slot, no contract edit per agent.
- **Other repos** → each gets its own `.siding.yml` contract; same `state.json` file keys by normalized repo root path across all projects on the machine.
- **Proxy on :1355** is implemented (`siding proxy up`). Caddy `http_port` is rewritten off `:80`; browsers use `http://app.<host>:1355`.
- **Agent isolation** is critical at 6+ parallel **local agents**: **one Cursor window, one folder** per agent — never a multi-root workspace bundling all clones. See below.
- **Local disk:** worktrees + dep symlinks reduce git/disk cost vs 6 full clones, but **isolation = one folder per window**, not clone vs worktree. Clones per local agent are fine if Tim prefers hard separation.
- **Port allocation:** `siding add` must scan all checkouts in one `state.json` before assigning — machine-global, not per-repo.

### Local agents only (not cloud)

Tim runs **local Cursor agents only** — no cloud agents. Every agent shares this machine’s filesystem; isolation is entirely a **workspace/window/folder** discipline.

**Current pattern:** each local agent gets its **own clone** (e.g. `pcaicode-jane`, `pcaicode-betty`). That works if each clone is opened in **its own single-folder Cursor window**.

**What failed before:** multiple clones/worktrees opened together in **one multi-root workspace** — local agents can read/write any root in that window. That is the likely cause of cross-agent contamination.

**Isolation that works for local agents:**

| Practice | Why |
|----------|-----|
| **One Cursor window → one folder** | Agent scope = that checkout only |
| **Agents window for monitoring** | OK to manage many agents; don’t use one window for all write access |
| **Git worktree per agent** (optional) | Same repo, less disk than 6 clones; still **one folder per window** |
| **Separate clone per agent** (Tim’s preference) | Hard separation; keep each in its own window |
| **New chat per task** | Reduces context bleed; does **not** replace folder isolation |

**Do not assume:** cloud VM isolation, `/in-cloud`, or that clones in a multi-root window are isolated from each other.

### Worktrees vs clones (local agents)

- Tim runs **~6 local agents** on parallel PosterChild features → up to **6 single-folder Cursor windows** when all are active
- Same repo, parallel branches → **worktrees + dep symlinks** save disk/git vs 6 full clones
- **Clone per local agent** is valid when each clone has **its own window** — not overkill for isolation; overkill only if all clones sit in one multi-root workspace
- Siding doesn’t care which; keys off **normalized repo root path**

---

## Key files map

```text
cmd/siding/main.go
internal/cli/                 # cobra commands (init, add, up/down/ps, plan, doctor, proxy)
internal/compose/             # host-side docker compose argv
internal/proxy/               # Caddyfile + siding-proxy-caddy
internal/contract/            # .siding/config.yml load/write, FindRepoRoot, stack from folder
internal/resolver/            # merge contract + state → plan result; slot env file
internal/state/               # state.json schema + store + port allocation
internal/paths/               # Abs, NormalizeLockPath, ResolvePaths
internal/doctor/
docker/Dockerfile.dev
docker/build.ps1
docker-compose.dev.yml
examples/hello/
```

---

## Agent instructions

1. **Read** `README.md`, `docs/CONTRACT.md`, `docs/STATE.md` for committed truth.
2. **Do not reintroduce** `main`, `default_stack`, workspace slugs, overlay beside exe, or a named `stacks:` roster in the contract.
3. **Minimize scope** — this is early scaffold; prefer small correct diffs.
4. **Verify:** `go test ./...` (via Docker if no local Go).
5. **Do not commit** this file — it is local handoff only.
6. If working on PosterChild integration later, reference `pcaicode-betty/docker/PCAI_CLI.md` and `docker/scripts/resolve-stack.mjs` — adapt to siding contract, don’t copy verbatim.

---

## Conversation origin

Distilled from design sessions Sep 2026: brand name **siding**, Go+GoReleaser, path-keyed state, removal of workspace slugs and `main`, Docker-based build/test without local Go, **local-only multi-agent** isolation (one window per folder, no cloud agents).
