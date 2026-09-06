# Operation reference

Every operation below is available two ways with identical arguments and results:

```sh
horde call OPERATION '{"json":"object"}'
```

or as an MCP tool on the `horde mcp` bridge. Arguments must be a JSON object.
Results are JSON.

Notation: `*` marks a required field. `outcome` identifies the task and is required
for every outcome-scoped operation when you call it from the CLI or the
personal-agent bridge. A worker token supplies `outcome` and `worker` itself and
rejects any attempt to name a different one.

## CLI shortcuts

| Command | Operation |
| --- | --- |
| `horde submit OBJECTIVE --repo P --template T` | `submit_outcome` |
| `horde list` | `list_outcomes` |
| `horde inspect ID` | `inspect` |
| `horde events ID --after N` | `events` |
| `horde metrics ID` | `metrics` |
| `horde cancel ID` | `cancel` |
| `horde resume ID` | `resume` |
| `horde answer ID QID ANSWER` | `answer_question` |
| `horde config get/set concurrency N` | `runtime_config_get` / `runtime_config_set` |
| `horde usage` | `account_status` |
| `horde runtime status/drain/resume/list/inspect/create/...` | `runtime_*` |

Other commands have no operation equivalent: `start`, `stop`, `daemon`, `mcp`,
`doctor`, `validate`, `config` (print), `service`, `update`, `network`.

## Task lifecycle

| Operation | Arguments | Notes |
| --- | --- | --- |
| `submit_outcome` | `objective`*, `repo`*, `template`, `context[]` | Returns `{"id":...}`. `repo` is an absolute path. `context` entries are context records. |
| `list_outcomes` | none | |
| `inspect` | `outcome`* | Tasks, attempts, workers, questions, integration state. |
| `events` | `outcome`*, `after`, `consumer` | Ordered activity events. |
| `ack_events` | `outcome`*, `consumer`*, `seq`* | Durable, monotonic per-consumer receipt. Does not answer questions. |
| `metrics` | `outcome`* | Usage, cost, latency, retries, coordination counts, `children` tree. |
| `cancel` | `outcome`* | Kills active process groups; cancels the owned subtree. |
| `resume` | `outcome`* | Interrupted processes must be reconciled first. |
| `add_tasks` | `outcome`*, `steps`* | Appends a validated workflow revision. |

## Questions

| Operation | Arguments | Worker |
| --- | --- | --- |
| `request_question` | `outcome`*, `worker`, `question`*, `id`, `evidence`, `recommendation`, `human_only` | yes |
| `pending_questions` | `outcome`* | yes |
| `answer_question` | `outcome`*, `question`*, `answer`*, `worker`, `human` | yes |
| `escalate_question` | `outcome`*, `question`*, `commentary`, `worker` | yes |

## Delegation

| Operation | Arguments | Worker |
| --- | --- | --- |
| `delegate_outcome` | `outcome`*, `id`*, `objective`*, `template`, `peer`, `bundles[]`, `worker` | yes |
| `list_children` | `outcome`* | yes |
| `integrate_child` | `outcome`*, `child`*, `validation[]`*, `worker` | yes |
| `read_context` | `outcome`*, `after`, `limit` | yes |
| `update_context` | `outcome`*, `content`*, `provenance`*, `kind`, `id`, `mandatory`, `supersedes[]` | no (root caller only) |

## Coordination

| Operation | Arguments | Worker |
| --- | --- | --- |
| `register_worker` | `outcome`*, `task` | no |
| `register_workspace` | `outcome`*, `worker`, `path`*, `branch`*, `base`* | yes |
| `list_workers` | `outcome`* | yes |
| `set_worker_status` | `outcome`*, `worker`, `status`* (`idle`/`working`/`blocked`/`stopped`) | yes |
| `claim_paths` | `outcome`*, `worker`, `paths[]`* | yes |
| `transfer_claim` | `outcome`*, `worker`, `to`*, `path`* | yes |
| `release_claims` | `outcome`*, `worker`* | no |
| `reconcile_worker` | `outcome`*, `worker`* | no |
| `send_message` | `outcome`*, `worker`, `id`*, `destination`*, `body`*, `refs{}`, `actionable` | yes |
| `read_messages` | `outcome`*, `worker`, `after`, `limit` | yes |
| `acknowledge_messages` | `outcome`*, `worker`, `ids[]`* | yes |
| `join_channel` | `outcome`*, `worker`, `channel`* | yes |
| `integrate` | `outcome`*, `worker`, `validation[]` | no |
| `propose_tasks` | `outcome`*, `worker`, `steps`* | yes (planner) |

## Artifacts and knowledge

| Operation | Arguments | Worker |
| --- | --- | --- |
| `put_artifact` | `outcome`*, `name`*, `content`*, `inputs{}`, `verified`, `worker`, `task` | yes, but cannot set `verified: true` |
| `get_artifact` | `outcome`*, `hash`* | yes |
| `reuse_artifact` | `outcome`*, `name`*, `inputs{}`* | yes |
| `add_knowledge` | `outcome`*, `kind`*, `content`*, `provenance{}`*, `inputs{}`, `verified`, `task` | yes, but cannot set `verified: true` |
| `knowledge` | `outcome`* | yes |
| `link_knowledge` | `outcome`*, `source`*, `target`*, `relation`* | yes |

## Environments and secrets

| Operation | Arguments | Worker |
| --- | --- | --- |
| `environments` | `outcome`* | yes |
| `refresh_bundles` | `outcome`* | no |

## Runtime and fleet management

None of these are available to a worker token.

| Operation | Arguments |
| --- | --- |
| `runtime_config_get` | none |
| `runtime_config_set` | `concurrency`* |
| `runtime_status` | none |
| `runtime_drain` / `runtime_resume` | none |
| `runtime_list` | none |
| `runtime_inspect` | `id`* |
| `runtime_create` | `id`*, `profile`*, `request_id`* |
| `runtime_destroy` / `runtime_restart` / `runtime_start` / `runtime_stop` | `id`*, `request_id`* |
| `runtime_update` | `id`*, `request_id`*, `version`* |
| `runtime_reconcile` | `id`*, `request_id`*, `resource`* |
| `runtime_updates_resume` | none |
| `account_status` | none |
| `account_observe` | `account`*, `provider`*, `window`*, `observed_at`*, `source`*, `used_percent`, `reset_at` |
| `management_events` | `after` |
| `management_ack` | `consumer`*, `seq`* |

## Worker token scope

A worker token may call exactly these, and nothing else:

`integrate_child`, `environments`, `delegate_outcome`, `list_children`,
`read_context`, `pending_questions`, `escalate_question`, `answer_question`,
`propose_tasks`, `request_question`, `send_message`, `read_messages`,
`acknowledge_messages`, `list_workers`, `set_worker_status`, `join_channel`,
`claim_paths`, `transfer_claim`, `register_workspace`, `put_artifact`,
`get_artifact`, `reuse_artifact`, `add_knowledge`, `knowledge`, `link_knowledge`.

Additional worker restrictions, enforced at the RPC layer:

- `outcome` and `worker` are forced to the token's own identity.
- A worker cannot attribute work to another worker's task.
- A worker cannot set `verified: true`. Workers report evidence; verification is
  an authoritative runtime decision.
- Arguments starting with `_` are rejected as reserved.

These are scope limits at the RPC layer, not process isolation. A worker token
does not protect against a malicious local process running as the same user.
