# uv-stack plugin

A Claude Code plugin carrying one skill, `using-uv-stack`, which teaches
coding agents to manage Python environments and project dependencies through
the [uv-stack](https://github.com/scott-arne/uv-stack) `stack` CLI
instead of ad-hoc `micromamba` / `uv pip compile` workflows.

The skill triggers on any Python environment or dependency work: adding or
removing packages, scaffolding projects, checking staleness or drift. Its
first step probes for the `stack` binary and stands aside gracefully on
machines where uv-stack is not installed.

## Contents

```
.claude-plugin/
  plugin.json           # plugin manifest
  marketplace.json      # single-plugin marketplace, source ./
skills/
  using-uv-stack/
    SKILL.md            # workflows, hard rules, common mistakes
    reference/
      commands.md       # full command/flag/file/env-var reference (v0.7.2)
```

## Installation

From GitHub:

```
/plugin marketplace add scott-arne/uv-stack-skill
/plugin install uv-stack@uv-stack
```

From a local checkout, replace the repository slug with the checkout path.
Verify with `/plugin` — the skill loads as `uv-stack:using-uv-stack`.

Prerequisites on the target machine: `uv`, `micromamba` (after
`micromamba shell init`), and `uv tool install uv-stack`.

## Machine-local configuration

The skill is generic; per-machine specifics live outside the plugin in a
single well-known file:

```
<config-root>/AGENTS.md
```

where the config root is resolved the same way `stack` resolves it:
`$UV_STACK_ROOT`, else legacy `$UV_ENV_ROOT`, else `~/.config/python-envs`.

If that file exists, the skill directs the agent to read it before acting and
to treat it as overriding the skill's generic defaults. Use it for policies
such as the machine's default shared environment, environments that must not
be modified, naming conventions, or local network quirks. Because it sits in
the config root, it travels with the environment definitions and is visible
to any agent runtime that honors the convention, not just Claude Code.
