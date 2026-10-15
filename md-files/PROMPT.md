# Prompt to paste into your coding agent

Build the `agy-swarm-bus` MCP plugin in this repository.

1. Read `SPEC.md` fully. It is the source of truth for architecture, schema, and tool behavior.
2. Follow `TASKS.md` in order. Tick each box as you finish it. Do not skip the tests.
3. Use TypeScript, compile to `dist/`, and use `better-sqlite3` plus `@modelcontextprotocol/sdk` plus `zod`.
4. Do not add features that are not in `SPEC.md`. If something is unclear, make the simplest choice and note it in `NOTES.md`.
5. When finished, run the tests and the smoke test, then print the final `mcp_config.json` snippet I should use (with absolute paths filled in).
