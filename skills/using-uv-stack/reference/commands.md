# stack command reference

Verified against uv-stack 0.4.4. `stack --help` and `<subcommand> --help` are
authoritative when versions differ.

## Contents

- Global (config-root resolution)
- Commands (full table with key options)
- Stack tokens (bare names, `profile:`, `@`/`bundle:`, `pkg:`, literals)
- Config root layout (directory tree, source vs generated files)
- Shared environments (upgrade semantics, conda-layer caveat, dry-run)
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

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack init` | Guided first-run setup (config tree, starter profile, first env) | `--yes` accepts defaults |
| `stack create env NAME [TOKENS]...` | Scaffold (with TOKENS) and build a shared environment | `--python VER` (writes `python.txt`, requires TOKENS), `--recreate` (wipe first), `--strict` |
| `stack create project TOKENS...` | Create a uv project in the current directory from resolved tokens | `--python SPEC-or-env-name`, `--name`, `--no-sync`, `--force` (add to existing pyproject), `--no-track`, `--strict` |
| `stack create profile NAME PKG...` | Write `profiles/NAME.yaml` | `--description`, `--tag` (repeatable) |
| `stack create bundle NAME TOKEN...` | Write `bundles/NAME.yaml` | `--description`, `--tag`, `--strict` |
| `stack upgrade [NAMES]...` | Render, compile, sync, check shared envs. No NAMES = all (prompts; `-y` skips) | `--all`, `--dry-run`, `--no-upgrade`, `--upgrade-package PKG` (repeatable), `--stop-on-error`, `--strict` |
| `stack refresh` | Re-resolve the tracked project in the current directory | `--dry-run` (shows add/remove delta), `--python` (override + record), `--no-sync`, `--strict` |
| `stack status [NAMES]...` | Per-env build state: exists, lock present, sources changed | `--json` |
| `stack list env\|profile\|bundle` | Tables of what exists | `--tag`, `--json` |
| `stack show env\|profile\|bundle [NAME]` | One item's details (env NAME defaults to `main`) | `--json` |
| `stack show project` | Tracked project here: tokens, applied packages, pending state | `--json` |
| `stack resolve TOKENS...` | Classify tokens; `--full` expands to a flat package list | `--full`, `--strict`, `--json` |
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
    ├── python.txt            # optional interpreter version (default 3.12)
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
- Batch upgrades continue past failures and end with a pass/fail summary;
  `--stop-on-error` aborts at the first failure.
- The lock is compiled to a temp file and atomically swapped, so a failed
  compile never corrupts the existing lock. If the sync step fails (e.g.
  network), rerun `stack upgrade NAME`.
- `--dry-run` runs no commands and never touches lock or env, but it DOES
  re-render `requirements.in` and `environment.yml`.

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
`stack upgrade`). It re-renders specs from config and checks lock freshness;
it does NOT compare installed packages against the lock. Installed-state
consistency is enforced at build time by `uv pip sync` + `uv pip check`.

It also cannot detect a live conda layer that predates edits to
`micromamba.txt`/`channels.txt`/`python.txt`: any later upgrade re-renders
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
