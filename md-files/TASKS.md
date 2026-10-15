# Build checklist (do in order)

- [ ] 1. `npm init`, install `@modelcontextprotocol/sdk better-sqlite3 zod` and dev deps `typescript vitest @types/node @types/better-sqlite3`.
- [ ] 2. `tsconfig.json` (target ES2022, module NodeNext, outDir `dist`, strict). Add scripts: `build`, `test`, `start`.
- [ ] 3. `src/db.ts`: open DB (absolute path, create dir), apply pragmas, create schema, export a `now()` helper and the path-normalization function.
- [ ] 4. Tools: agents (`register_agent`, `list_agents`), locks (`acquire_file_lock`, `release_file_lock`, `release_all_locks`), mail (`send_peer_message`, `poll_mailbox`), tasks (`create_task`, `list_tasks`, `claim_and_update_task`).
- [ ] 5. `src/index.ts`: create the MCP server over stdio, register all tools with zod schemas, generate `session_id` with `crypto.randomUUID()`, start cleanup timer, handle shutdown signals.
- [ ] 6. Tests in `test/` for each acceptance criterion in `SPEC.md`. For the multi-process lock test, open two separate `better-sqlite3` connections to the same temp file.
- [ ] 7. Smoke test: build, run the server, and send a `tools/list` request over stdio to confirm all tools appear.
- [ ] 8. Write `mcp_config.example.json` with a placeholder absolute path and `README.md` with setup steps (build, copy `AGENTS.md` into the workspace, add MCP config).
- [ ] 9. Final check: run `npm test`, confirm nothing writes to stdout except protocol output.
