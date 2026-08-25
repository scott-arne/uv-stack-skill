# stack command reference

Verified against uv-stack 0.4.5 (`stack --version` reports the installed
version). `stack --help` and `<subcommand> --help` are authoritative when
versions differ.

## Contents

- Global (config-root resolution)
- Commands (grouped as in `stack --help`, with key options)
- Stack tokens (bare names, `profile:`, `@`/`bundle:`, `pkg:`, literals)
- Config root layout (directory tree, source vs generated files)
- Shared environments (upgrade semantics, changing Python, conda-layer caveat, dry-run)
- Projects (tracking, interpreter resolution, refresh ownership, pending state)
- What `stack status` covers (and its blind spots)
- Environment variables

## Global

```
stack [--root PATH] COMMAND ...
```

Config root resolution: `--root` > `$UV_STACK_ROOT` > `$UV_ENV_ROOT` (legacy)
> `~/.config/python-envs`.

## Commands

Grouped as in `stack --help`.

### Create

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack create env NAME [TOKENS]...` | Scaffold (with TOKENS) and build a shared environment; without TOKENS, retarget an existing env's interpreter | `--python VER` (writes `python.txt`; without TOKENS it requires `--recreate`), `--recreate` (compile the lock first, then wipe and rebuild), `--strict` |
| `stack create project TOKENS...` | Create a uv project in the current directory from resolved tokens | `--python SPEC-or-env-name`, `--name`, `--no-sync`, `--force` (add to existing pyproject), `--no-track`, `--strict` |
| `stack create profile NAME PKG...` | Write `profiles/NAME.yaml` | `--description`, `--tag` (repeatable) |
| `stack create bundle NAME TOKEN...` | Write `bundles/NAME.yaml` | `--description`, `--tag`, `--strict` |

### Environments

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack upgrade [NAMES]...` | Render, compile, sync, check shared envs. No NAMES = all (prompts; `-y` skips) | `--all`, `--dry-run`, `--no-upgrade`, `--upgrade-package PKG` (repeatable), `--stop-on-error`, `--strict` |

### Projects

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack refresh` | Re-resolve the tracked project in the current directory | `--dry-run` (shows add/remove delta), `--python` (override + record), `--no-sync`, `--strict` |

### Inspection

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack status [NAMES]...` | Per-env build state: exists, lock present, sources changed, Python changed | `--json` |
| `stack list env\|profile\|bundle` | Tables of what exists | `--tag`, `--json` |
| `stack show env\|profile\|bundle [NAME]` | One item's details (env NAME defaults to `main`) | `--json` |
| `stack show project` | Tracked project here: tokens, applied packages, pending state | `--json` |
| `stack resolve TOKENS...` | Classify tokens; `--full` expands to a flat package list | `--full`, `--strict`, `--json` |

### Maintenance

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack init` | Guided first-run setup (config tree, starter profile, first env) | `--yes` accepts defaults |
| `stack doctor` | Detect problems, print `fix:` suggestions; never changes anything without `--fix` | `--fix` (safe repairs only), `--json` |
| `stack completion bash\|zsh\|fish` | Shell completion script | |
| `stack config init` | Create missing config directories (bare primitive; `stack init` is the guided form) | |

## Stack tokens

| Token | Meaning |
| --- | --- |
| `ds` (bare) | Profile `ds` if it exists, else bundle `ds`, else literal package |
| `profile:ds` | Profile, error if missing |
| `@standard` / `bundle:standard` | Bundle, error if missing |
| `pkg:numpy` / `package:numpy` | Always a literal package (escape hatch for name collisions) |
| `numpy>=2`, `-e ~/src/mytool`, archive paths | Literal pip requirement, passed through |

Near-miss bare tokens trigger a "did you mean" warning; `--strict` (on
`upgrade`, `create env/project/bundle`, `resolve`, `refresh`) turns any
unqualified fallthrough into an error.

## Config root layout

```
<config-root>/
├── project-python.txt        # optional: default --python for `create project`
├── profiles/<name>.yaml      # description, tags, includes: [pip requirements]
├── bundles/<name>.yaml       # includes: [stack tokens]; bundles can nest
├── .locks/                   # internal lock files — leave alone
└── envs/<name>/
    ├── stack.txt             # REQUIRED; one token per line; # comments
    ├── python.txt            # optional interpreter version (default 3.12); applied at create/--recreate only
    ├── micromamba.txt        # optional extra conda packages
    ├── channels.txt          # optional extra conda channels (conda-forge always first)
    ├── requirements.local.in # optional machine-local pip additions
    ├── requirements.in       # GENERATED
    ├── environment.yml       # GENERATED
    └── requirements.lock.txt # GENERATED (uv pip compile)
```

## Shared environments

- Enter: `micromamba activate NAME`; one-off: `micromamba run -n NAME CMD`.
- `stack upgrade NAME` default recompiles with `--upgrade` (all pins float),
  then syncs the env exactly to the lock — packages removed from sources are
  uninstalled. `--no-upgrade` re-locks without floating pins (smallest safe
  change when adding); `--upgrade-package PKG` floats one.
- Conda-layer sources (`micromamba.txt`, `channels.txt`, `python.txt`) render
  into `environment.yml`, but a plain `stack upgrade` of an existing env runs
  micromamba only if the env is missing — the live conda layer is applied at
  creation or `--recreate` only. After editing them, rebuild with
  `stack create env NAME --recreate`.
- Changing an env's interpreter: `stack create env NAME --python 3.14
  --recreate`. `--recreate` requires `python.txt` to hold a plain dotted
  version (a match spec such as `3.12.*` is refused, because the lock is
  resolved against that value with `uv pip compile --python-version`); a
  match spec is still legal when creating with TOKENS and no `--recreate`.
  The candidate lock is compiled *before* `micromamba remove` runs, so an
  unsatisfiable resolve leaves the old environment intact, and the lock is
  published only after the rebuild succeeds.
- A non-recreate `stack upgrade` refuses outright when the env's running
  interpreter no longer satisfies `python.txt`, rather than syncing the pip
  layer onto the wrong interpreter. The error names both remedies: edit
  `python.txt` back to the running version, or recreate. The probe fails
  open — an unavailable micromamba or an unparseable version never
  manufactures a refusal.
- Batch upgrades continue past failures and end with a pass/fail summary;
  `--stop-on-error` aborts at the first failure.
- The lock is compiled to a temp file and atomically swapped, so a failed
  compile never corrupts the existing lock. If the sync step fails (e.g.
  network), rerun `stack upgrade NAME`.
- `--dry-run` runs no mutating commands and never touches lock or env, but it
  DOES re-render `requirements.in` and `environment.yml`, and it DOES issue
  one read-only interpreter probe — a dry run is held to the same
  version-drift refusal as the real thing, so the plan it prints never
  describes commands the real run would decline to issue.

## Projects

- `stack create project` runs `uv init --bare`, `uv add`, `uv sync`, and (by
  default) records tokens in `[tool.uv-stack]` in `pyproject.toml`
  (`--no-track` for one-shot scaffolds).
- `--python`: a version/path/spec like `cpython@3.12` passes straight to uv;
  **anything else is treated as a micromamba env name** and the project uses
  that environment's interpreter. Default: `$UV_STACK_PROJECT_PYTHON`, then
  `<config-root>/project-python.txt`, then 3.12.
- `stack refresh` removes only packages in the table's `applied` list (names
  uv-stack itself added); other dependencies are never touched. Ownership is
  by package name: re-pin a stack-applied package yourself and a later
  refresh may rewrite or remove it.
- `[tool.uv-stack]` is tool-owned and rewritten wholesale (comments in that
  table are not preserved); every other byte of `pyproject.toml` is left
  alone — though refresh runs `uv remove`/`uv add` under the hood, which edit
  `[project.dependencies]` normally.
- Interrupted run: a `pending` key remains in the table; `stack show project`
  reports it. The next successful `stack refresh` (or re-running the tracked
  create with `--force`) cleans it up. Recovery adopts leftover pending names
  still installed but no longer in the stack — it warns first, and the next
  refresh removes them (the warning says how to keep one). There is no
  auto-resume.
- Day to day a tracked project is still a normal uv project: `uv add`,
  `uv sync`, `uv run` all work.

## What `stack status` covers

Per environment: does the micromamba env exist, is a lock present, did
sources change since the last build ("sources changed" means run
`stack upgrade`), and does the running interpreter still match `python.txt`.
It re-renders specs from config and checks lock freshness; it does NOT
compare installed packages against the lock. Installed-state consistency is
enforced at build time by `uv pip sync` + `uv pip check`.

States, in the precedence order they are decided: `config error`,
`not created`, `never built`, `python changed`, `sources changed`,
`lock stale`, `ok` — so a drifted interpreter masks a `sources changed` row
until it is resolved.

The probe (`micromamba run -n NAME python -c ...`) reports both versions:
the table shows `3.12 (env 3.14)` in the Python column for a drifted env,
and `--json` carries `actual_python` beside `python` (null when the probe
could not run). It fails open — an unavailable micromamba, a failed
`python -c`, or an unparseable version leaves the state alone rather than
reporting false drift, as does a `python.txt` that is not a plain dotted
version (nothing meaningful to compare). A configured `3.14` is satisfied by
an actual `3.14.7`: the comparison is component-wise, only as far as
`python.txt` specifies.

It still cannot detect a live conda *package* layer that predates edits to
`micromamba.txt`/`channels.txt`: any later upgrade re-renders
`environment.yml`, so status returns to "ok" while the installed conda layer
still lacks the change. When those files changed after the env was built,
compare `micromamba list -n NAME` against the env's `environment.yml`; fix
with `stack create env NAME --recreate`.

## Environment variables

| Variable | Purpose |
| --- | --- |
| `UV_STACK_ROOT` | Config root (overridden by `--root`) |
| `UV_ENV_ROOT` | Legacy spelling; used only when `UV_STACK_ROOT` unset/empty |
| `UV_STACK_PROJECT_PYTHON` | Default interpreter spec for `create project` |
| `MAMBA_EXE` | micromamba binary path; set by `micromamba shell init`, preferred over PATH |

For workflow guidance and hard rules, see [../SKILL.md](../SKILL.md).
