# stack command reference

Verified against uv-stack 0.6.0 (`stack --version` reports the installed
version). `stack --help` and `<subcommand> --help` are authoritative when
versions differ.

## Contents

- Global (config-root resolution, NAME rules)
- Commands (grouped as in `stack --help`, with key options)
- Stack tokens (bare names, `profile:`, `@`/`bundle:`, `pkg:`, literals, `${NAME}`)
- Config root layout (directory tree, source vs generated files)
- Shared environments (upgrade and sync semantics, changing Python, conda-layer caveat, dry-run)
- Projects (tracking, interpreter resolution, refresh and sync project, pending state)
- Editing config files (`stack edit`, editor precedence)
- Comparing environments (`stack diff` sources, layers, verdicts)
- Portable config roots (what travels, bring-up, `.gitignore`, variables)
- What `stack status` covers (and its blind spots)
- Environment variables

## Global

```
stack [--root PATH] COMMAND ...
```

Config root resolution: `--root` > `$UV_STACK_ROOT` > `$UV_ENV_ROOT` (legacy)
> `~/.config/python-envs`.

A NAME given to `create`, `edit`, `show`, `status`, `upgrade`, or `sync env`
is a file stem: one holding a path separator, a `.` or `..` segment, `:`, `@`,
whitespace, or a leading `-` is refused. A hand-made directory with such a
name still shows in `stack list env` and a bare `stack status`; rename it on
disk to address it by name.

## Commands

Grouped as in `stack --help`.

### Create

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack create env NAME [TOKENS]...` | Scaffold (with TOKENS) and build a shared environment; without TOKENS, retarget an existing env's interpreter | `--python VER` (writes `python.txt`; without TOKENS it requires `--recreate`), `--recreate` (compile the lock first, then wipe and rebuild), `--strict` |
| `stack create project TOKENS...` | Create a uv project in the current directory from resolved tokens | `--python SPEC-or-env-name`, `--name`, `--no-sync`, `--force` (add to existing pyproject), `--no-track`, `--strict` |
| `stack create profile NAME PKG...` | Write `profiles/NAME.yaml` | `--description`, `--tag` (repeatable) |
| `stack create bundle NAME TOKEN...` | Write `bundles/NAME.yaml` | `--description`, `--tag`, `--strict` |

### Edit

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack edit env\|profile\|bundle\|project [NAME]` | Open a config file in an editor, validate it when the editor exits, re-offer on error. `env` NAME defaults to `main`; `project` takes no NAME and edits `./pyproject.toml`. Never creates anything | `--file stack\|python\|micromamba\|channels\|local` (env only; default `stack`; a missing optional file is opened anyway), `--editor CMD` |

### Environments

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack upgrade [NAMES]...` | Render, compile, sync, check existing shared envs. No NAMES = all (prompts; `-y` skips). Never creates a missing env | `--all`, `--dry-run`, `--no-upgrade`, `--upgrade-package PKG` (repeatable), `--stop-on-error`, `--strict` |

### Projects

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack refresh` | Re-resolve the tracked project in the current directory | `--dry-run` (shows add/remove delta), `--python` (override + record), `--no-sync`, `--strict` |

### Sync

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack sync` | Create, build, and recompile every env the root declares. Creates missing envs, never prompts, preserves existing pins | `--dry-run`, `--stop-on-error`, `--strict`, `--upgrade` (force new pins) |
| `stack sync env NAMES...` | The same, for the named envs only | same as `stack sync` |
| `stack sync project TOKENS...` | Append TOKENS to the tracked project's recorded stack and re-resolve (a token already recorded makes it a plain refresh) | `--dry-run`, `--python`, `--no-sync`, `--strict` |

### Inspection

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack status [NAMES]...` | Per-env build state: exists, lock present, sources changed, Python changed | `--json` |
| `stack list env\|profile\|bundle` | Tables of what exists | `--tag`, `--json` |
| `stack show env\|profile\|bundle [NAME]` | One item's details (env NAME defaults to `main`) | `--json` |
| `stack show project` | Tracked project here: tokens, applied packages, pending state | `--json` |
| `stack resolve TOKENS...` | Classify tokens; `--full` expands to a flat package list | `--full`, `--strict`, `--json` |
| `stack diff SOURCE SOURCE` | Compare two environments' declared config and resolved pins (see Comparing environments) | `--json`, `--exit-code` |

### Maintenance

| Command | Purpose | Key options |
| --- | --- | --- |
| `stack init` | Guided first-run setup (config tree, starter profile, first env) | `--yes` accepts defaults |
| `stack doctor` | Detect problems, print `fix:` suggestions; never changes anything without `--fix`. Also checks portability: declared variables with no value, bad `${...}` references, missing editable checkouts, a `project-python.txt` that will not travel, a missing or stale managed `.gitignore` block, a name that is both a profile and a bundle | `--fix` (safe repairs only), `--json` |
| `stack completion bash\|zsh\|fish` | Shell completion script | |
| `stack config init` | Create missing config directories (bare primitive; `stack init` is the guided form) | |
| `stack config portable` | Write the managed `.gitignore` block and `.gitkeep` placeholders in empty top-level dirs; print the git commands to run next. Never runs git | `--dry-run` (show block and next steps; write nothing) |

## Stack tokens

| Token | Meaning |
| --- | --- |
| `ds` (bare) | Profile `ds` if it exists, else bundle `ds`, else literal package |
| `profile:ds` | Profile, error if missing |
| `@standard` / `bundle:standard` | Bundle, error if missing |
| `pkg:numpy` / `package:numpy` | Always a literal package (escape hatch for name collisions) |
| `numpy>=2`, `-e ~/src/mytool`, archive paths | Literal pip requirement, passed through |

Near-miss bare tokens trigger a "did you mean" warning; `--strict` (on
`upgrade`, `sync`, `create env/project/bundle`, `resolve`, `refresh`) turns
any unqualified fallthrough into an error.

Entries in profiles, bundles, and `stack.txt` may reference `${NAME}` for a
variable the root declares (see Portable config roots). An entry spanning
more than one line or ending in a trailing backslash is refused by every
command that renders.

## Config root layout

```
<config-root>/
├── variables.txt             # optional: declares the ${NAME}s entries may reference
├── variables.local.txt       # optional: this machine's NAME=value lines; not committed
├── .gitignore                # managed block written by `stack config portable`
├── project-python.txt        # optional: default --python for `create project`
├── editor.txt                # optional: editor command for `stack edit`; not committed
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
  uninstalled. `--no-upgrade` re-locks keeping the pins the existing lock
  holds (smallest safe change when adding); `--upgrade-package PKG` floats
  only PKG. Before 0.6.0 both flags silently re-resolved everything, so the
  first run under 0.6.0 may move pins that should already have stayed put.
- `stack sync` / `stack sync env NAMES...` do the same render-compile-sync
  but create missing envs instead of failing on them, never prompt, and
  preserve existing pins by default (`--upgrade` floats them). Upgrade is
  for envs that exist; sync is for "make this root real on this machine".
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
- Upgrade and sync batches continue past failures and end with a summary of
  succeeded, failed, and skipped envs (skipped appears only under
  `--stop-on-error`, which aborts at the first failure). A `--dry-run` batch
  prints the summary too.
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
- `stack sync project TOKENS...` appends TOKENS to the recorded stack
  (`["standard"]` + `@qsar` becomes `["standard", "@qsar"]`) and re-resolves
  through the same recovery as refresh. Use it to add tokens; use
  `stack refresh` to re-resolve the recorded ones. Neither removes a token —
  to drop one, edit `stack` in `[tool.uv-stack]` (or `stack edit project`),
  then `stack refresh`.
- `stack create project` and `stack refresh` warn when the interpreter spec
  they record will not resolve on another machine (an absolute path, or a
  micromamba env only this machine has).
- Interrupted run: a `pending` key remains in the table; `stack show project`
  reports it. The next successful `stack refresh` (or re-running the tracked
  create with `--force`) cleans it up. Recovery adopts leftover pending names
  still installed but no longer in the stack — it warns first, and the next
  refresh removes them (the warning says how to keep one). There is no
  auto-resume.
- Day to day a tracked project is still a normal uv project: `uv add`,
  `uv sync`, `uv run` all work.

## Editing config files

- `stack edit` is interactive: it blocks on the editor, validates the file
  when the editor exits, and on error shows it and offers the editor again
  with the changes in place. Nothing is reverted; declining leaves the file
  as edited and exits non-zero. With stdin not a terminal there is no
  re-offer (error, exit 1); an editor exiting non-zero (`:cq`) aborts
  without validating.
- On success it names how to apply: `stack upgrade NAME` for an env,
  `stack refresh` for a project, next upgrade/refresh for a profile or bundle.
- Editor: `--editor` > `$UV_STACK_EDITOR` > `<config-root>/editor.txt` >
  `$VISUAL` > `$EDITOR`; empty values are skipped; none set = refuses. The
  value is a command line (`code -w`, `emacsclient -nw`).
- A GUI editor that returns immediately (`code`/`subl` without `-w`, `gvim`
  without `-f`) makes validation run before any edit and report success.
  Always use the blocking flag.
- `editor.txt` strips everything from `#`, so an editor command containing
  `#` must come from an environment variable.
- Symlinked config files work (the success line names the real target); a
  dangling link is refused.

## Comparing environments

`stack diff SOURCE SOURCE`. A source is an env in this root, the path to
another machine's `envs/<name>/` directory wherever it sits (it is read in
place, never copied into this root), or a path to a
`requirements.lock.txt`.

- Four layers: interpreter (`python.txt`), micromamba packages, effective
  channel order, compiled pins. A bare lock carries pins only; the other
  three are reported as not compared, never as matching.
- The first three are compared as *declared*: two envs that both say `3.12`
  can run different patch releases. Only the pins are *resolved*. `diff`
  reads what was compiled, never what is installed.
- Verdict: `identical`, `identical-where-comparable` (pins matched, a bare
  lock hid the rest), or `different`. Exit 0 for all three; `--exit-code`
  exits 1 on `different`. An unreadable source exits 1 either way.
- Compiled locks do not travel with a cloned root, so comparing against
  another machine means copying its `envs/<name>/` directory (lock included)
  here first.

## Portable config roots

A config root can live in git and be cloned elsewhere.

| Travels | Stays machine-local |
| --- | --- |
| `profiles/`, `bundles/`, `envs/*/stack.txt`, `python.txt`, `micromamba.txt`, `channels.txt` | `requirements.in`, `environment.yml`, `requirements.lock.txt` (generated) |
| `variables.txt` (declared names) | `variables.local.txt` (this machine's values) |
| `project-python.txt` when it holds a version, a uv form (`cpython@3.12`), or an env this root declares | `envs/*/requirements.local.in`, `editor.txt`, `.locks/` |

- **Locks deliberately do not travel**: each machine compiles for its own
  platform, so machines built from the same sources get compatible envs,
  not identical pins. `stack diff` shows the difference.
- **Bring-up on a new machine:** clone, `stack doctor` (names declared
  variables with no value here), fill `variables.local.txt`, `stack sync`.
- **Making a root committable:** `stack config portable` writes a block
  between `# BEGIN uv-stack` / `# END uv-stack` markers in `<root>/.gitignore`
  (lines outside it are preserved; inside is replaced on each run), adds
  `.gitkeep` to empty `profiles/`, `bundles/`, `envs/`, and prints the git
  commands to run — including `git rm --cached` for already-tracked files
  that are now ignored. It never runs git. It refuses a `.gitignore` that is
  a symlink or has a second hard link. A root inside a larger repo (dotfiles)
  works: the printed commands use `git -C <root>`. Its printed `commit`
  records everything already staged in that repo, so check `git status`
  first.
- **Variables:** `variables.txt` lists names, one per line (`#` comments).
  `variables.local.txt` holds `NAME=value` lines. An exported environment
  variable of a declared name wins over the file (how CI supplies values);
  undeclared names are never read from the environment. Reference as
  `${NAME}` in profiles, bundles, `stack.txt`; expanded into
  `requirements.in` and `uv add`, but a tracked project's `[tool.uv-stack]`
  keeps the unexpanded entry.
- Variable values: no whitespace (a path with a space needs a symlink); a
  value containing `${` is refused, not expanded (substitution is
  single-pass). `stack doctor` reports both. A `#` in the file form is
  silently stripped as a comment; export the variable instead.
- The limit: `uv add` writes a local source's (editable or path) resolved
  absolute path into the project's own `[tool.uv.sources]`, so a tracked
  project that pulls in a local source is machine-bound in that table.

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
| `UV_STACK_EDITOR` | Editor for `stack edit`; beats `editor.txt`, `$VISUAL`, `$EDITOR` |
| `VISUAL`, `EDITOR` | Fallback editors for `stack edit`, in that order, below `editor.txt` |
| any name in `variables.txt` | This machine's value for `${NAME}`; beats `variables.local.txt` |
| `MAMBA_EXE` | micromamba binary path; set by `micromamba shell init`, preferred over PATH |

For workflow guidance and hard rules, see [../SKILL.md](../SKILL.md).
