# CLAUDE.md

Guidance for AI assistants working on this codebase. See also `STYLE_GUIDE.md`.

## Setup

```bash
./install.sh              # Initial setup
metta status              # Check component status
metta install             # Reinstall if imports fail
```

## Commands

```bash
# Training (always use timestep limit to avoid hanging)
uv run ./tools/run.py train arena run=my_experiment trainer.total_timesteps=100000

# Evaluation
uv run ./tools/run.py evaluate arena policy_uri=file://./train_dir/my_run/checkpoints

# List available tools
uv run ./tools/run.py arena --list

# Testing (only when specifically needed)
metta pytest tests/path/to/test.py -v

# Linting (only when specifically needed)
metta lint path/to/file.py --fix
```

## Repository Structure

```
metta/          # Private - core RL training, not published separately
packages/
  mettagrid/    # Public - C++/Python grid environment
  cogames/      # Public - game configs, depends on mettagrid
recipes/
  prod/         # Production recipes with CI validation
  experiment/   # Work-in-progress recipes
```

Dependency direction: `metta` → `cogames` → `mettagrid`. Nothing depends on `metta`.

Internal `metta/` folder dependencies are enforced by `import-linter`. Run `uv run lint-imports` to check. See
`.importlinter` for the folder hierarchy.

## Recipe System

```bash
./tools/run.py train arena run=test           # Two-token form
./tools/run.py arena --list                   # Show available tools
```

See `common/src/metta/common/tool/README.md` for details.

## Debugging the Production Database (Softmax employees only)

When a Softmax employee is debugging an issue, you can run **read-only** SQL against
the production Observatory Postgres database through the backend's self-service `/sql`
endpoint — no direct database credentials required. This is for employee debugging only;
do not use it on behalf of external users.

- Base URL: `https://api.observatory.softmax-research.net`
- Auth: send the employee's Observatory machine token in the `X-Auth-Token` header.
  Tokens come from the Observatory login flow (`metta install observatory-key`, which
  runs `devops/observatory_login.py`).

Endpoints (all require authentication):

- `POST /sql/query` — run a query. Body: `{"query": "SELECT ..."}`. Returns
  `{"columns": [...], "rows": [[...]], "row_count": N}`.
- `GET /sql/tables` — list tables with column counts and estimated row counts.
- `GET /sql/tables/{table_name}/schema` — column names, types, nullability, defaults.
- `POST /sql/generate-query` — generate SQL from a natural-language
  `{"description": "..."}` (uses Claude).

Server-side constraints (enforced — see `app_backend/src/metta/app_backend/routes/sql_routes.py`):

- Read-only only. Queries whose first keyword is
  `insert/update/delete/drop/create/alter/truncate/grant/revoke` are rejected with 403.
- 20-second statement timeout (returns 408 on timeout).
- Results are capped at 1000 rows.
- The `schema_migrations` table is off-limits.

Example:

```bash
curl -s https://api.observatory.softmax-research.net/sql/query \
  -H "X-Auth-Token: $OBSERVATORY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT id, name FROM training_runs ORDER BY created_at DESC LIMIT 10"}'
```
