# Retrospective: Task CRUD HTTP Endpoints

**Date:** 2026-05-15
**Task:** 73d2f8cd-d01f-4ab6-a023-2b3005e93d94
**Agent:** Coding Tester

## What Went Well
- **Explore agent for upfront research** — Delegating a thorough codebase exploration at the start gave me a complete map of endpoints, auth mechanics, request/response shapes, and SQL schemas before writing a single curl. This prevented guessing and made test design fast once the environment was stable.
- **Comprehensive endpoint coverage** — Once the environment was stable, testing all 30 cases (CRUD, lifecycle, dependencies, validation, auth, concurrency) went quickly and produced clear, tabular evidence.
- **Concurrency test for TOCTOU concern** — Firing 5 parallel dispatches and reading server logs to confirm exactly 1 success / 4 conflicts was a simple, effective way to verify the reviewer's race condition concern without needing a dedicated test harness.
- **Cross-workspace isolation test** — Creating a second workspace and attempting cross-workspace task access directly verified the workspace scoping requirement with real DB state.

## What Was Painful
- **Docker container restarting and stealing port 8080** — ~15 minutes lost. The `multica-backend-1` Docker container had a restart policy that kept bringing it back after I stopped it. Each restart replaced my freshly-built server binary with the old Docker image (which lacked the PR's routes). Required `docker update --restart=no` plus `kill -9` to fully reclaim the port. This happened at least 3 times during the session. Affects any agent or developer who needs to run a local server binary while Docker Compose services are defined.
- **Three different database connections** — ~10 minutes lost. The environment had three potential Postgres targets: (1) `DATABASE_URL` env var pointing to `postgres:postgres@localhost:5432/sb`, (2) `.env` file specifying `multica:multica@localhost:5433/multica`, (3) Docker container's internal DB. I created test data in the wrong database first (Docker's port 5433), then discovered the server was reading from the env var's database (port 5432). No documentation or error message clarified which database the running server was actually using.
- **JWT secret discovery** — ~10 minutes lost across 4 attempts. The `.env` file said `JWT_SECRET=change-me-in-production`, the Go code had a hardcoded default `multica-dev-secret-change-in-production`, and the actual runtime environment had `leafy-jwt-secret-key-20250513` (set by the CI/task harness). Each wrong secret produced a generic "invalid token" error with no hint about which secret was expected. I had to read `/proc/<pid>/environ` to find the truth.
- **Server process churn** — ~10 minutes lost. The server died or was replaced at least 4 times during testing (Docker restarts, env var changes requiring restarts, port conflicts). Each restart required regenerating JWT tokens and re-verifying connectivity. The HTTP 000 responses from curl (connection refused) were initially confusing because they looked like routing failures rather than server crashes.
- **JWT token expiration between generation and use** — ~5 minutes lost. Tokens generated with `go run` in one shell invocation expired or were invalid by the time a later batch of curl commands ran. The `go run` compilation time (~3-5 seconds) plus sequential test execution meant tokens were sometimes stale. I had to inline token generation at the top of each test batch.
- **`go` not in PATH** — ~3 minutes lost. The Go binary was at `/usr/local/go/bin/go` but not in the default PATH. Every shell invocation needed `export PATH="/usr/local/go/bin:$PATH"`. The Makefile handles this internally, but direct `go` commands fail silently with "command not found".

## What I'd Do Differently
- **Kill Docker Compose services first, unconditionally.** Before starting any local server binary, run `docker compose stop && docker update --restart=no` on all services. Don't assume `docker stop` is sufficient.
- **Read `/proc/<pid>/environ` immediately** when encountering auth failures, instead of guessing JWT secrets from config files. The running process's environment is the single source of truth.
- **Use a single test script with token generation inlined** rather than generating tokens in one command and using them in another. Token freshness is critical when the JWT has a short TTL.
- **Verify DATABASE_URL from the running server's perspective first** before creating any test data. A 30-second check of the process environment would have saved 10 minutes of wrong-database debugging.
- **Set a longer JWT expiry (48h+) for test tokens** to eliminate token-expiration-during-testing as a failure mode entirely.

## Recommendations
- **Add a `make server-local` target** that (1) stops Docker backend, (2) sets restart=no, (3) sources the correct env, (4) runs the Go binary. This would eliminate the port-conflict and env-var-mismatch issues that consumed ~25 minutes of this session. Affects all agents and developers running local server binaries.
- **Log the JWT secret source on startup** (e.g., `INF jwt secret source=env_var` or `source=default`). A single log line would save minutes of debugging when tokens are rejected. The current "invalid token" error gives no hint whether the problem is the secret, the signing method, or the expiry.
- **Log the DATABASE_URL (redacted) on startup.** The server already logs pool config but not which database it connected to. Adding the host:port/dbname to the "connected to database" log line would make wrong-database issues immediately obvious.
- **Document the test environment setup** for agents: which database, which JWT secret, how to create test users/workspaces. The handler tests use `TestMain` with fixtures, but there's no equivalent guide for manual/API-level testing.
