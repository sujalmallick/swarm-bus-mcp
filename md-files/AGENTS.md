# Swarm rules (copy this file into the root of the user's workspace)

Several agent tabs may be working on this project at the same time. They coordinate through the `swarm-bus` MCP server. Follow these rules in every task.

## At the start of every task
1. Call `register_agent` with your role (for example `backend-agent` or `frontend-agent`) and a one-line description of your task.
2. Call `poll_mailbox` and read every message before doing anything else.
3. Call `list_tasks`. If your work matches an existing task, claim it with `claim_and_update_task(task_id, "in_progress")`. If `claimed` is false, pick different work.

## Before editing any file
1. Call `poll_mailbox` again (it is cheap).
2. Call `acquire_file_lock(filepath)`.
   - If `acquired` is false, do NOT edit that file. Work on something else, or wait and retry later.
3. Make your edit.
4. Call `release_file_lock(filepath)` as soon as you are done with that file. Do not hold locks across long pauses.

## When you change something others depend on
If you change an API shape, a shared type, a config key, a database column, or a file path other agents use, call `send_peer_message(target_role, payload)` right away. Say what changed, in which file, and what the other side must update. Use `"*"` to notify everyone.

## When you finish
1. Mark your task `done` with `claim_and_update_task`.
2. Call `release_all_locks`.
3. Send a short summary message to any role affected by your work.

## Never
- Never edit a file you do not hold the lock for.
- Never ignore a message from `poll_mailbox`. If it affects your work, adjust before continuing.
- Never bypass the swarm tools because a task "looks small".
