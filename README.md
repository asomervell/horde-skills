# Horde skills

[![skills.sh](https://skills.sh/b/asomervell/horde-skills)](https://skills.sh/asomervell/horde-skills)

Agent skills for [Horde](https://horde.sh), a local Rust daemon that assigns
software tasks to coding agents and keeps running after your session ends.

```sh
npx skills add asomervell/horde-skills
```

| Skill | For |
| --- | --- |
| **`horde`** | An agent using Horde. Why it exists and when not to use it, installation, MCP wiring, the submit and monitor loop, answering questions, collecting results, crash recovery. References cover setup, configuration, tasks, delegation, app environments, GitHub delivery, networking and managed runtimes, and the complete operation API. |
| **`horde-templates`** | Authoring the versioned TOML workflow templates Horde compiles into a task graph: step fields, substitution, parallel write scopes, bounded repair loops, nested templates, role fallback. |
| **`horde-worker`** | An agent running inside a Horde task: read context first, register a workspace, claim paths before writing, exchange mail, raise durable questions, record artifacts, delegate within bounds. |

Install one of them on its own:

```sh
npx skills add asomervell/horde-skills --skill horde
```

## What Horde is

An agent session is ephemeral, single-threaded, and forgetful. Horde is a local
daemon that owns scheduling instead. You submit an objective and a repository; it
expands that into a durable workflow, runs coding agents in isolated Git worktrees,
records every attempt, and integrates the result onto a separate branch.

- **Durable.** SQLite in WAL mode with FULL synchronous writes. Disconnecting a
  client cancels nothing.
- **Safely parallel.** Each worker gets its own worktree and must claim the paths
  it edits. Overlapping claims are rejected with ownership evidence.
- **Honest after a crash.** An interrupted model call, shell command, merge, or
  GitHub write is marked uncertain and held for explicit reconciliation, never
  silently replayed.
- **Multi-model.** Roles map to executors: Codex CLI, Claude Code CLI, a Tuara API
  model, or a simulated no-op, with configured fallback on capacity.
- **Bounded.** Delegation trees are capped, and the original intent and constraints
  travel down to every child.

Install Horde itself with:

```sh
curl -fsSL https://horde.sh/install | bash
```

Documentation lives at [horde.sh/docs](https://www.horde.sh/docs).

## Contributing

These files are mirrored from the Horde source repository. Open an issue here for
anything inaccurate, and include the Horde version from `horde --version`.

Apache-2.0 licensed.
