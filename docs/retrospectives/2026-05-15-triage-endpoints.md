# Retrospective: Triage Proposal & Finalize Endpoints

**Date:** 2026-05-15
**Task:** d21d440f-2220-4207-9f26-67903b6dbbc1
**Agent:** Coding Tester

## What Went Well
- **Comprehensive acceptance criteria coverage** — All 6 acceptance criteria were verified with 31 discrete checks against a live server and real PostgreSQL. No mocks, no test framework shortcuts. This gave high confidence in the implementation.
- **Database-level verification** — Every API response was cross-checked against direct DB queries (triage_proposal rows, task rows, task_dependency rows, workspace.task_counter). This caught the full transaction path, not just response shaping.
- **Subagent delegation for code exploration** — Using the Explore agent to read all 8 changed files in parallel saved significant context window space and gave a complete picture of the PR before any testing began.
- **Auth mechanism discovery** — The Explore agent efficiently mapped the JWT generation, auth middleware, and test fixture patterns, enabling curl-based testing without needing to reverse-engineer the login flow.

## What Was Painful
- **Stale server process on port 8080** — A pre-existing `/tmp/multica-server` binary (from a prior `make start`) was occupying port 8080 with a different JWT_SECRET. This caused silent auth failures (401) that looked like JWT generation bugs. **~10 minutes wasted** diagnosing token encoding (compact JSON, spaces in claims) when the real issue was a different server answering requests. **Affects all testers** using this environment.
- **Docker port mapping mismatch** — The `.env` file specifies `POSTGRES_PORT=5432` but the Docker container maps 5432 internally to 5433 on the host. The server startup failed with "role does not exist" — a misleading error that looks like a missing DB user rather than a wrong port. **~5 minutes wasted.** **Affects any agent or developer** starting the server manually outside of `make start`.
- **Migration 076 already-applied error** — The migration had been partially applied (table and column existed, migration record existed), but `go run ./cmd/migrate up` still tried to re-run it and failed on `column already exists`. The migrator doesn't gracefully handle this state. **~3 minutes wasted.** **Affects anyone** rebasing or rerunning migrations on a branch where the schema was already applied.
- **Server process dying between requests** — The Go server started successfully but shut down silently between test runs (visible in logs as `INF shutting down server`). This required three restart cycles. Root cause unclear — possibly the background process received a signal, or the shell session cleanup killed it. **~8 minutes wasted** across multiple restarts and re-verification.
- **Go not on PATH in `make` context** — `go` was available at `/usr/local/go/bin/go` but not on the default shell PATH used by `make`. `make setup` failed at the migration step with `/bin/sh: go: not found`. Had to manually export PATH for all Go commands. **~3 minutes wasted.** **Affects all Go operations** in this environment.
- **Test script debugging** — The first bash test script used a `jq_val` helper with Python `eval()` that silently failed on dictionary key access. Had to rewrite the entire script with inline `python3 -c` calls and `<<< "$BODY"` heredocs. **~5 minutes wasted.** This is a self-inflicted issue, not an infrastructure problem.

## What I'd Do Differently
- **Kill all processes on port 8080 before starting** — Always run `lsof -ti:8080 | xargs kill -9` as the first step, before even building the server. The stale process was the single biggest time sink.
- **Detect the actual Postgres port from Docker** — Instead of trusting `.env`, run `docker port multica-postgres-1 5432` to get the real host-side port, then construct DATABASE_URL dynamically.
- **Use a simpler test harness** — Write the test script in Python instead of bash+python3 hybrid. The bash heredoc + python parsing was fragile and hard to debug. A single Python script with `requests` would have been cleaner and worked on the first try.
- **Check server health in a loop after starting** — Instead of `sleep 3 && curl`, use `until curl -sf http://localhost:8080/api/config; do sleep 1; done` with a timeout. This would catch server crashes immediately.
- **Pin the JWT secret early** — Generate the JWT once, verify it works with `/api/me`, and save both the token and the server PID. If any subsequent request returns 401, immediately check if the PID is still alive before debugging the token.

## Recommendations
- **Fix: Add `POSTGRES_PORT` auto-detection to worktree/manual startup docs** — The `.env` says 5432 but Docker maps to 5433. Either the Makefile should detect this, or `.env` should be generated from the actual Docker port mapping. This is a recurring trap for any non-`make dev` startup path.
- **Fix: `make dev` / `make start` should kill stale server processes** — Before starting a new server, check if port 8080 is already occupied and warn or kill the stale process. The current behavior silently fails with `bind: address already in use` buried in logs.
- **Improve: Migration idempotency** — The custom migrator should handle the case where the schema changes exist but the migration record is missing (or vice versa) more gracefully than a hard `column already exists` error. At minimum, a clearer error message pointing to the resolution.
- **Nice-to-have: Health endpoint** — Add a `/health` or `/api/health` endpoint that returns 200 (currently returns 404). This would make startup detection and monitoring simpler. The existing `/api/config` works as a proxy but isn't semantically correct.
