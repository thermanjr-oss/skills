---
name: cli-anything
description: Guide to CLI-Anything (HKUDS/CLI-Anything), which generates stateful, agent-friendly Python CLIs for software that has source code so agents can drive real apps without GUI automation. Use when the user wants to make an app agent-controllable, build or refine a CLI harness, install a ready-made cli-anything-<software> CLI from CLI-Hub, or run /cli-anything, refine, test, or validate.
---

# Working With CLI-Anything

[CLI-Anything](https://github.com/HKUDS/CLI-Anything) (Apache-2.0) is a methodology plus tooling that generates a "harness": a Click-based Python CLI for an application, so an agent can operate it through commands instead of clicking through a GUI. Generated CLIs have a REPL (the default when run with no subcommand), a `--json` flag for machine-readable output, and undo/redo.

Decide first: does a CLI for this app already exist? If so, install it (below). If not, and you have the app's source, generate one.

## Install a ready-made CLI

```bash
pip install cli-anything-hub
cli-hub search <query>
cli-hub install <name>
```

Other hub commands: `list`, `info <name>`, `update <name>`, `uninstall <name>`, `launch <name>`. Requires Python 3.10+. On Windows, Git for Windows or WSL is needed.

Per-app skills for other agents: `npx skills add HKUDS/CLI-Anything --skill cli-anything-<software> -g -y`.

## Generate a CLI (Claude Code plugin)

```
/plugin marketplace add HKUDS/CLI-Anything
/plugin install cli-anything
```

Manual install: clone the repo, copy `cli-anything-plugin` to `~/.claude/plugins/cli-anything`, then `/reload-plugins`.

| Command | Use |
| --- | --- |
| `/cli-anything <path-or-repo>` | Full build. Example: `/cli-anything ./gimp` |
| `/cli-anything:refine <path> [focus]` | Gap analysis, then adds commands, tests, and docs. Incremental and non-destructive, safe to repeat |
| `/cli-anything:test <path-or-repo>` | Run tests and update TEST.md |
| `/cli-anything:validate <path-or-repo>` | Check against HARNESS.md standards |
| `/cli-anything:list [--path <dir>] [--depth <n>] [--json]` | List installed and generated tools |

If `/cli-anything` isn't found, try `/cli-anything:cli-anything`. OpenCode and Cursor use hyphenated names (`/cli-anything-refine`). Codex, Hermes, and Reasonix take plain language ("Use CLI-Anything to refine ./shotcut").

## The build phases

1. **Codebase analysis**: backend engine, data model, GUI-to-API mapping. Output is an SOP doc such as `GIMP.md`.
2. **CLI architecture**: command groups, JSON state model, output formats.
3. **Implementation**: Click CLI, REPL, `--json`, undo/redo.
4. **Test planning**: TEST.md with a unit and E2E plan.
5. **Test implementation**: `test_core.py`, `test_full_e2e.py`, subprocess tests.
6. **Test documentation**: run `pytest -v --tb=no` and append results to TEST.md.
6.5. **SKILL.md generation**: writes `skills/cli-anything-<software>/SKILL.md`.
7. **Publish**: `setup.py`, `pip install -e .`, check `which cli-anything-<software>`.

Output layout:

```
skills/cli-anything-<software>/SKILL.md
<software>/agent-harness/
  <SOFTWARE>.md, setup.py
  cli_anything/            # PEP 420 namespace, no __init__.py here
    <software>/            # has __init__.py, __main__.py, <software>_cli.py
      core/  utils/  tests/
```

Install and check a generated CLI:

```bash
cd <software>/agent-harness && pip install -e .
cli-anything-<software> --help
```

## Driving a generated CLI as an agent

- Always pass `--json` and parse that, not the human output.
- Check the exit code and read stderr on failure.
- Use absolute paths.
- Verify output files exist after the command; don't trust the success message alone.
- State lives in the project/session, so run related commands against the same project file.

## Caveats

- It needs the target app's source code. For closed-source apps, check the hub for an existing harness instead.
- This skill summarizes the README and plugin README. The exact phase text lives in `cli-anything-plugin/HARNESS.md` upstream; read it before customizing a build.
- Generated harnesses wrap real applications. Review what a command does before running it on files that matter.
