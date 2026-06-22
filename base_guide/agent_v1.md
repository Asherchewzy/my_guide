## Scope & PR Discipline

- Target ~15 files changed per session. If the natural scope exceeds this, flag it and suggest splitting the work before writing any code.
- Before writing code, list the files you expect to touch. If the list feels large, say so.
- Features are product concepts — break them into independently mergeable engineering units. Each session should target one unit.
- Refactors and feature work should not be mixed. If you spot something worth refactoring while implementing a feature, note it in a comment (`# TODO: refactor candidate — <reason>`) and move on.
- Aim for the smallest change that moves the codebase to a valid, non-broken state.

---

## Module Design (after Ousterhout)

- **Prefer deep modules over shallow ones.** A module should hide significant complexity behind a simple interface. Avoid splitting logic into many small functions or classes just for the sake of it — each abstraction should earn its existence.
- **Different layer, different abstraction.** Each layer (e.g. repository, service, route) should have its own vocabulary. Don't pass raw DB models up to the API layer, and don't leak HTTP concepts down into business logic.
- **Define errors out of existence where possible.** Prefer designs that make invalid states unrepresentable over scattered defensive checks. When errors must exist, handle them at the right layer — not everywhere.
- **General-purpose over special-purpose.** When writing a module, ask whether a slightly more general version would serve multiple use cases without added complexity. Avoid one-off abstractions.


### Design review checklist
- Will this make future changes easier or harder?
- Does it create hidden dependencies?
- Does it obscure important behavior?
- Is a small change going to require edits in many places later?
- Is the module deep?
- Is information hidden in one place?
- Are there pass-through layers?
- Are errors defined out of existence where reasonable?
- Are comments useful and current?

---

## Brainstorming protocol
When asked to design or implement a non-trivial change, produce a short design note before editing.


**Use this format internally and, when helpful, share it with the developer:**


```
Goal:
- What behavior needs to exist?

Existing design:
- Which modules currently own related knowledge?
- What patterns already exist?

Complexity risks:
- Change amplification:
- Cognitive load:
- Unknown unknowns:
- Dependencies:
- Obscurity:

Candidate designs:
- Option A:
  Interface:
  Hidden knowledge:
  Pros:
  Cons:
- Option B:
  Interface:
  Hidden knowledge:
  Pros:
  Cons:

Choice:
- Pick the design with the best long-term simplicity.
- Explain why.

Verification:
- Tests to run:
- Edge cases:
```

---

## Comments & Documentation

- Comments should explain _why_, not _what_. The code says what; the comment explains the reasoning, tradeoffs, or constraints that aren't obvious from reading it.
- Don't comment things that are self-evident from the code.
- Document interfaces, not implementations. A function's docstring should describe its contract — inputs, outputs, side effects — not narrate the body.

---

## FastAPI (Backend)

- Follow a controller / service / repository folder structure. Keep route handlers thin — business logic belongs in the service layer.
- Services should not import from routes. Repositories should not import from services.
- Use Pydantic models for all request/response shapes. Don't pass raw dicts across layer boundaries.
- Prefer explicit dependency injection via `Depends()` over module-level globals.

---

## React + Vite + TanStack Start (Frontend)

- Co-locate components with the routes that use them unless they're genuinely reusable.
- Use TanStack Start server functions (`createServerFn`) for server-only logic (DB access, secrets). All other components are standard React — there is no server/client component split.
- Fetch data via route `loader` + React Query `queryOptions` for SSR-hydrated pages. Use `apiFetch` (in `lib/api/`) for client-initiated requests. Avoid prop-drilling fetched data — co-locate queries with the routes or components that consume them.
- Keep API calls in a dedicated layer (e.g. `lib/api/`), not scattered across components.

---

## Testing

- Tests are part of the implementation, not an afterthought. Write them in the same session unless explicitly told otherwise.
- **Test behaviour, not implementation.** Tests should assert on inputs and outputs, not on how the internals work. If refactoring breaks a test without changing observable behaviour, the test was wrong.
- **Test at the right layer.** Unit test services and pure logic. Integration test repositories and API routes. Don't unit test things that are better covered by integration tests, and don't write integration tests for things that are trivially covered by unit tests.
- One test file per module. Keep test structure parallel to source structure (e.g. `tests/services/test_user_service.py` mirrors `app/services/user_service.py`).
- Prefer real objects over mocks where it's not expensive. Mock at boundaries — external APIs, third-party services, the filesystem — not between internal layers.
- Each test should have one reason to fail. Avoid asserting on multiple unrelated behaviours in a single test.
- Tests should be readable as documentation. A test name should describe the scenario and expected outcome: `test_create_user_returns_409_when_email_already_exists`, not `test_create_user_error`.
- **Python services: run tests via `uv run pytest`**, not bare `pytest` or `python -m pytest`. This ensures tests execute inside the project-managed venv with all declared dependencies resolved. From the service directory:

  ```bash
  uv run pytest tests/unit/ -v
  ```

- **Python services: two testing modes.** Use `uv run pytest tests/unit/ -v` (fast local) during iteration. Run `make test` (Docker) as the final verification before opening a PR. Docker is the canonical path that matches CI.
- **Python services: use mocks and fixtures, not `monkeypatch`.** Define reusable state in `pytest` fixtures (in `conftest.py` or the test module). Mock external boundaries with `unittest.mock` or a library like `respx`. `monkeypatch` is a pytest escape hatch for modifying globals/env — prefer designs that don't require it.

---

## Error Handling

- **Handle errors at the right layer, not everywhere.** Repositories surface DB exceptions. Services translate them into domain errors. Routes translate domain errors into HTTP responses. Don't catch and re-raise through every layer.
- Prefer domain-specific exceptions over generic ones. `UserNotFoundError` is more useful than `ValueError("user not found")`.
- In FastAPI, use exception handlers registered at the app level for consistent HTTP error shaping. Don't write `try/except` blocks in route handlers unless the handling is genuinely route-specific.
- In the frontend, errors from `lib/api/` should be typed and handled at the route or component level — not swallowed silently or logged to console only.
- Never expose internal error details (stack traces, DB messages) in API responses. Log them server-side; return a clean, structured error shape to the client.
- Distinguish between expected errors (user input, not found, conflict) and unexpected errors (infra failure, unhandled exception). Handle them differently — expected errors are part of the domain, unexpected errors should alert.

---

## Naming

- Names should be precise enough that a reader doesn't need to look at the implementation to understand the intent. If you find yourself writing a comment to explain what a variable holds, the name is probably wrong.
- Avoid generic names: `data`, `result`, `info`, `handler`, `manager`, `util`. Name things by what they specifically represent.
- Functions should be named for what they return or what they do — not how they do it. `get_active_users()` over `query_users_with_active_flag()`.
- Boolean variables and functions should read as assertions: `is_expired`, `has_permission`, `can_publish` — not `check_expiry` or `expiry_status`.
- Be consistent with the codebase's existing vocabulary. If the codebase calls it a `policy`, don't introduce `rule` or `regulation` for the same concept.
- When a name feels hard to choose, that's often a signal the abstraction itself isn't clear yet. Pause and reconsider the design before forcing a name.

---

## Git Commit Granularity

- Commit messages should complete the sentence: _"This commit will..."_ — e.g. `add user authentication middleware`, not `auth stuff` or `wip`.
- Don't bundle a refactor and a feature in the same commit. If a refactor was necessary to enable a feature, commit the refactor first, then the feature.
- Avoid committing commented-out code, debug logs, or `print` statements.
- If a session produces work that spans multiple logical changes, flag it and suggest how to split the commits before finishing.

---

## Pull Request Guidelines

When creating a PR, follow this template:

`.github/pull_request_template.md`

PR title should include the Jira ticket prefix, e.g. `[VIRES-123]: add pagination for chat log retrieval`.

---

## General

- When in doubt between two approaches, briefly note the tradeoff and your reasoning before implementing. Don't just pick one silently.
- Consistency with existing patterns in the codebase takes precedence over personal preference.
- Don't introduce a new dependency without flagging it. Prefer solving problems with what's already in the stack.

---

## Tools used within this repo

### `frontend/`

- **Package manager:** pnpm (`pnpm@10.32.1`) — use `pnpm` for all Node commands, never `npm` or `yarn`
- **Runtime requirement:** Node >=22
- **Lock file:** `pnpm-lock.yaml`
- Install: `pnpm install`
- Dev server: `pnpm dev`
- Build: `pnpm build`
- Test: `pnpm test` (Vitest) / `pnpm test:watch`
- Lint: `pnpm lint` / `pnpm lint:fix`
- Coverage: `pnpm coverage`

### `backend/`

- **Package manager:** uv (Python 3.13)
- **Lock file:** `uv.lock`
- Install: `uv sync`
- Test (unit, fast): `uv run pytest tests/unit/ -v`
- Test (canonical / CI): `make test`
- Lint: `make lint` (ruff + black)
- Lint-fix: `make lint-fix`

### `orchestrator/`

- **Package manager:** uv (Python >=3.10)
- **Lock file:** `uv.lock`
- Install: `uv sync`
- Test (unit): `make test-unit`
- Test (canonical / CI): `make test`
- Lint: `make lint` / `make lint-fix`

### `mcp-enterprise-search/`

- **Package manager:** uv (Python >=3.13)
- **Lock file:** `uv.lock`
- Install: `uv sync`
- Test (unit): `make test-unit`
- Test (canonical / CI): `make test`
- Lint: `make lint` / `make lint-fix`

### Root

- `make docker-up` / `docker-down` / `docker-build` — start/stop/rebuild all services via Docker Compose
- `make lint` / `make test` — runs lint/test across all Python services

---

## Pre-PR Checklist

Before creating or pushing to a PR branch, run lint and format checks locally so CI doesn't fail on avoidable issues.

**Python services** (from the service directory, e.g. `backend/` or `mcp-enterprise-search/`):

```bash
make lint           # ruff check + black --check

# Auto-fix if needed:
make lint-fix       # ruff check --fix + black (applies all fixes)
```

Or directly with uv if no Makefile target exists:

```bash
uv run ruff check src/ tests/
uv run black --check src/ tests/

# Auto-fix if needed:
uv run ruff check --fix src/ tests/ && uv run black src/ tests/
```

**Frontend** (from `frontend/`):

```bash
pnpm lint
```

If either step reports errors, fix them and add a commit before pushing. Don't suppress linter rules — fix the underlying issue.

---

## Proposed Updates

If you encounter a pattern, decision, or constraint that isn't covered above, propose an addition to this document rather than introducing an undocumented convention.