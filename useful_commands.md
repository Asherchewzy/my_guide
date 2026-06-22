# Useful Command Cheat Sheet

## Load Environment Variables
```bash
export $(grep -v '^#' .env.example | xargs)
```

### Verify important variables
```bash
printenv | grep -E 'API_BASE_URL|DATABASE_URL|AUTH_CLIENT_ID|API_KEY'
```

---

## UV Python Environment

### Create a `uv` virtual environment
```bash
cd backend
uv venv
```

### Create a `uv` virtual environment with a specific Python version
```bash
cd backend
uv venv --python 3.12
```

### Activate the environment
```bash
source .venv/bin/activate
```

### Install a package for local use only
```bash
uv pip install <package>
```

Use this when you want the package in the current environment without updating `pyproject.toml` or `uv.lock`.

### Install a package for your computer
```bash
uv tool install <package>
```

Use this for CLI tools you want available outside a single project.

### Add a package to the project
```bash
uv add <package>
```

This updates `pyproject.toml`, refreshes `uv.lock`, and installs the dependency into the project environment.

### Add a development dependency
```bash
uv add --dev <package>
```

This also updates `pyproject.toml` and `uv.lock`, but marks the package as a dev dependency.

### Sync the environment from the lockfile
```bash
uv sync
```

Use `uv sync` when you want the environment to match `uv.lock` exactly. It does not change `pyproject.toml`.

### `uv add` vs `uv sync`

- `uv add <package>` changes project dependencies and updates `pyproject.toml` and `uv.lock`.
- `uv sync` installs what is already defined in `uv.lock` into the environment.
- Use `uv add` when you are introducing or changing dependencies.
- Use `uv sync` when you are setting up a fresh clone or making the environment match the lockfile.


---

## Authenticate with Cloud Provider 
### AWS
```bash
aws sso login --profile meow
```

### Azure
```bash
az login
```

### GCP
```bash
gcloud auth login
```

---

## Useful git commands

### personal excludes
```bash
# Add files here
Vi .git/info/exclude
```

### Rebase Branch

```bash
git fetch origin
git checkout <your-branch>
git rebase origin/main
```

### Resolve conflicts
```bash
git add .
git rebase --continue
```

### Update remote branch
```bash
git push --force-with-lease
```


## Docker Services

### Rebuild and start
```bash
docker compose up --build
```

### Start in background
```bash
docker compose up -d
```

### Stop services
```bash
docker compose down
```

### Rebuild from scratch
```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```

### Clean up
```bash
docker system prune -f
docker system prune -a --volumes -f
docker builder prune -f
```
---



---

## Run Frontend Locally

```bash
cd frontend
pnpm install
pnpm dev
```

---


## Database Migrations

### View current migration
```bash
alembic current
```

### View migration history
```bash
alembic history --verbose
```

### Roll back one revision
```bash
alembic downgrade -1
```

### Upgrade to latest
```bash
alembic upgrade head
```

### Stamp database without running migrations
```bash
alembic stamp head
```

---

## Run Tests
add `uv run` infront of commands 

### All tests
```bash
make test
```

### Unit pytests 
```bash
pytest tests/unit/ -v
```

### Specific test file
```bash
pytest tests/unit/test_service.py -v
```

### Specific test
```bash
pytest tests/unit/test_service.py::test_create_user -v
```

---

## Type Checking

### Python
```bash
mypy src/
```

### TypeScript
```bash
pnpm tsc --noEmit
```

---

## Linting & Formatting
lints to consider: black, isort, ruff, pydocstyle --convention=google, pre-commit 

### Python
```bash
ruff check src/ tests/
black --check src/ tests/
```

### Auto-fix
```bash
ruff check --fix src/ tests/
black src/ tests/
```

### JavaScript / TypeScript
```bash
pnpm lint
pnpm prettier --check .
```

### Auto-format
```bash
pnpm prettier --write .
```

---


---



---
