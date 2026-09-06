---
name: horde
description: Run long, parallel, or risky coding work on Horde, a local daemon that assigns software tasks to coding agents and survives your session ending. Use when a task needs to keep running after you exit, needs several agents editing one repository at once without collisions, needs a plan/implement/review pipeline, needs a different model than the one you are running on, or needs a durable record of what each attempt did. Also use to install Horde, connect it over MCP, configure executors and concurrency, submit and monitor tasks, answer agent questions, collect results from worktrees, and recover after a crash. Triggers include "horde", "horde.sh", "run this in the background", "delegate this task", "run these in parallel", "keep working while I'm gone", "hand this to another agent".
---

# Horde

Horde is a local Rust daemon that owns scheduling for coding work. You submit an
objective and a repository; Horde expands it into a durable workflow, runs coding
agents in isolated Git worktrees, records every attempt, and integrates the result
onto a separate branch. It keeps running after the client that submitted the work
exits.

You keep your own agent. Horde does not replace this session. It is the place you
put work that should outlive it.

## Why it exists

An agent session is ephemeral, single-threaded, and forgetful. That is fine for a
small edit and wrong for everything else. Horde fixes five specific failures:

1. **Work dies with the session.** Horde's state is SQLite in WAL mode with FULL
   synchronous writes. The daemon owns the schedule. Disconnecting a CLI or MCP
   client does not cancel anything.
2. **Parallel agents corrupt each other.** Every worker gets its own worktree and
   must acquire an exclusive claim on the paths it edits. Overlapping claims are
   rejected with ownership evidence. Integration into the shared result is
   serialized per outcome.
3. **A crash silently replays side effects.** Horde never assumes an interrupted
   model call, shell command, merge, or GitHub write did nothing. It marks the
   attempt uncertain, blocks the outcome, and requires explicit reconciliation.
4. **One model does everything.** Roles (`planner`, `worker`, `reviewer`, and any
   role you name) each map to an executor: Codex CLI, Claude Code CLI, a Tuara
   API model, or a simulated no-op. Mix them per step, with configured fallback
   when an account runs out of capacity.
5. **Agents guess instead of asking.** A worker can raise a durable question that
   holds only its own task. The question travels up the caller chain, unchanged,
   to you.

Everything Horde reports is evidence-backed. A child reporting success is
provisional until the parent imports its commits and runs combined validation.

## When to use it, and when not to

Use Horde when the work is long-running, needs more than one agent, needs a
verification pass by a different model, needs to survive a restart, or needs an
auditable record of attempts.

Do not use Horde for a change you can make in this session in a couple of edits.
Do not use it as a security sandbox: it is a cooperative, single-user runtime, not
an isolation boundary against hostile code running as the same user. Do not expect
a dashboard, a distributed scheduler, a hosted service, or a team-shared instance.

## Setup

Check for an existing installation before installing anything:

```sh
horde --version || outcome --version
```

If `outcome` is present but `horde` is not, this is a pre-rename installation.
Run `~/.local/bin/outcome update` once, then use `horde`. Existing configuration,
data, and service identity are reused. Do not create empty `~/.config/horde` or
`~/.local/share/horde` directories: legacy paths are only used when the Horde
equivalents do not exist.

Fresh install (macOS or Linux; requires Git):

```sh
curl -fsSL https://horde.sh/install | bash
horde start
```

The installer verifies the signed release before writing anything and asks whether
to start Horde at login. In a non-interactive shell it installs only the binary
unless you pass `--service`.

Then verify the whole scheduling path without spending a single model call:

```sh
cd /path/to/a/git/repo
horde submit "Exercise the runtime" --repo . --template simulated
```

If that returns an id and `horde inspect ID` shows tasks completing, Horde works.
Only then configure a real executor. See `references/setup.md` for MCP wiring,
boot services, data directories, and troubleshooting.

## Connect Horde to this agent

Add a stdio MCP server so you can submit and monitor without shelling out:

```json
{
  "mcpServers": {
    "horde": { "command": "horde", "args": ["mcp"] }
  }
}
```

This bridge is administrative. Never hand it to an untrusted worker; workers get
a separate, restricted bridge with a scoped token.

Everything the MCP bridge exposes is also available as `horde call OPERATION 'JSON'`
with identical arguments and results, so a shell is always a valid fallback. Every
CLI command prints pretty JSON. Parse it; do not scrape it.

## The loop

```sh
horde submit "Add CSV export with tests" --repo /path/to/repository   # -> {"id":"..."}
horde inspect TASK_ID      # tasks, attempts, workers, questions, integration
horde events TASK_ID       # ordered activity, use --after SEQ to tail
horde metrics TASK_ID      # tokens, cost, latency, retries, coordination counts
```

Rules that matter:

- The repository must have an initial commit and a configured Git author.
- Coding happens in separate worktrees. Your checkout stays on its branch.
- The integrated result lands on branch `outcome/TASK_ID`. Find it with
  `git worktree list`. Diff it before you trust it.
- Nothing is pushed and no PR is opened unless delivery is explicitly enabled.
- Settings are pinned at submission. Editing config later affects new tasks only.

When a worker needs a decision, `horde inspect` shows a pending question:

```sh
horde answer TASK_ID QUESTION_ID "Use CSV and omit identifying fields"
```

Answer within your authority or, if you are an intermediate caller, escalate the
original envelope one level with `escalate_question`. Never rewrite a question
into a different question.

To stop or restart work:

```sh
horde cancel TASK_ID
horde resume TASK_ID
```

## After a crash

A hard daemon crash marks running attempts uncertain and blocks their outcomes.
This is deliberate. Do not try to force it forward.

1. `horde inspect TASK_ID` to find the uncertain attempt and its worker.
2. Inspect that worker's worktree and any external effects (pushed branches, open
   PRs, running app processes).
3. Stop orphaned processes. `reconcile_worker` refuses while a recorded process is
   still alive.
4. `horde call reconcile_worker '{"outcome":"TASK_ID","worker":"WORKER_ID"}'`
5. `horde resume TASK_ID`

Claims survive the crash and stay with their owner until reconciled and released.

## Where the rest lives

Read the reference only when you need it.

| Need | File |
| --- | --- |
| Install, MCP wiring per agent, boot service, data dirs, migration, troubleshooting | `references/setup.md` |
| Settings TOML, executor roles, auth modes, credential broker, concurrency, limits | `references/configuration.md` |
| Task lifecycle, results, revisions, artifacts, knowledge, recovery in depth | `references/tasks.md` |
| Bounded delegation trees, inherited context, question routing, child acceptance | `references/delegation.md` |
| App `.env` bundles, disposable process and Compose test environments | `references/environments.md` |
| GitHub push, PR, checks, merge, deployment, health delivery | `references/delivery.md` |
| Tailscale/mTLS networking, managed E2B/Daytona/Docker/Kubernetes runtimes, updates | `references/fleet.md` |
| Every operation, its arguments, and whether a worker token may call it | `references/operations.md` |

Related skills: `horde-templates` to author workflow templates, `horde-worker`
for an agent running as a worker inside a Horde outcome.

## Non-negotiables

- Never invent a task, worker, question, or artifact id. Read it from output.
- Never report a task as done because a child or worker said so. Confirm through
  `inspect` and, for delegated work, through `integrate_child` with real validation.
- Never bypass reconciliation after a crash or an uncertain attempt.
- Never put provider API keys in an app secret bundle. Provider credentials belong
  in executor configuration; bundles are for the software being built.
- Never share the personal-agent MCP bridge with a worker.
- Set provider API keys in the **daemon's** environment before `horde start`.
  Worker command environments use an allowlist that omits them.
- Report what the evidence shows, including unknown cost and unknown capacity.
  Unknown is not zero.
