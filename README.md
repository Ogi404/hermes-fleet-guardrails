# Hermes fleet guardrails

Safety controls for running a team of AI agents on the open-source [Hermes agent framework](https://github.com/NousResearch/hermes-agent). They decide whether an agent is allowed to start a task, and whether a person has approved the exact code that is about to be pushed or merged.

This repository is an extract from my working fork of Hermes. It holds only the files I added, so it is a code sample to read, not a standalone package.

## The problem

Once several agents share a task board, the dangerous failures are quiet ones:

- An agent starts work on a task whose instructions were edited after they were approved.
- A hand-written card imitates an approved task and gets dispatched.
- A builder agent picks up a task meant for a reviewer.
- A person approves a change in chat, the code changes afterwards, and the old approval is applied to the new code.

Each control below exists to turn one of those into a clear, recorded rejection.

## What's here

| Part | File | What it does |
|---|---|---|
| Contract guard | `hermes_cli/kanban_contract_guard.py` | Checks every fleet task against a frozen, hashed copy of its instructions before an agent may start it |
| Project registry | `hermes_cli/kanban_fleet_project_registry.py` | Resolves which project and working folder a task truly belongs to |
| Human gate | `hermes_cli/kanban_human_gate.py` | Records a person's approve, hold or reject decision against one exact commit |
| Lifecycle controller | `plugins/fleet_lifecycle/` | An opt-in operator tool for staging, planning and verifying agent work (see its own README for the detailed protocol) |
| Tests | `tests/` | 328 tests covering the above |

## How the contract guard works

When a task is created from an approved issue, its instructions are frozen into a snapshot inside the task, together with a SHA-256 hash of the fields that matter. Before any agent is dispatched, the guard re-checks the task and returns either nothing (go ahead) or a short rejection code.

| If this happens | The guard returns |
|---|---|
| The snapshot's content no longer matches its hash | `snapshot_hash_mismatch` |
| The task title was changed after approval | `task_title_mismatch` |
| A card carries only one of the two provenance markers | `provenance_mismatch` |
| The snapshot markers are missing, duplicated or reversed | `snapshot_marker_invalid` |
| The assigned agent isn't allowed in this task's lane, or is a control-plane agent | `project_role_permission_forbidden` |
| The task's project doesn't match the repository in the contract | `repository_project_mismatch` |
| The working folder isn't the task's own worktree | `task_workspace_path_invalid` |
| The source issue is marked blocked or not ready for agents | `snapshot_blocked`, `snapshot_not_agent_ready` |

Two design choices matter most:

- **It fails closed.** Anything the guard can't validate is rejected.
- **It never raises.** Task text is outside input. A malformed task returns `snapshot_invalid` and is set aside on its own; it can't crash the dispatcher and stall every other task on the board.

Ordinary cards that aren't fleet tasks pass through untouched.

## How the human gate works

A gate request sends a small summary to the chat where the work was requested. The decision that comes back is one of approve, hold or reject, at one of three stages: push, ready or merge.

Every decision is bound to the full 40-character commit SHA it was made against and stored as an event on the task. If the code changes after the request, the earlier approval doesn't apply to the new commit. Gate changes from non-admin users are refused before they reach the dispatcher. No language model is involved in any of this.

## Running the tests

The tests need a Hermes checkout, because these files extend Hermes modules.

```bash
git clone https://github.com/Ogi404/hermes-agent
cd hermes-agent
git checkout fleet/held-slack-brief-intake-20261116
pip install ruamel.yaml httpx pytest pytest-asyncio
python -m pytest \
  tests/hermes_cli/test_kanban_fleet_contract_guard.py \
  tests/hermes_cli/test_kanban_fleet_project_registry.py \
  tests/hermes_cli/test_kanban_human_gate.py \
  tests/gateway/test_kanban_human_gate_access.py \
  tests/gateway/test_fleet_controller_failure_notifier.py \
  tests/plugins/test_fleet_*.py -q
```

Result on that branch: 328 tests. Run as a normal user, they are expected to pass. Run as root, 326 pass and 2 fail on purpose, because the verifier refuses to run as root.

## Status and limits

- This code lives in my fork and is not part of upstream Hermes.
- The lifecycle controller is opt-in and is not activated by default.
- The tests run offline against temporary databases and simulated services. They show the controls behave as specified; they are not a measure of production reliability.
- Project names, issue numbers and paths in the test fixtures and documentation are neutral samples.

## How it was built

I built this with AI coding tools, mainly Codex and Claude. The requirements, the design decisions, the review and the responsibility for what runs are mine.

## Licence

MIT, the same licence as Hermes. Hermes is copyright Nous Research; see `LICENSE`.

## Contact

Ognen Djabirski · [Portfolio](https://ogi404.github.io/portfolio/) · [LinkedIn](https://www.linkedin.com/in/ognen-djabirski)
