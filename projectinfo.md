# DataForge OpenEnv Project Info

Source: `resources/dataforge_complete_steps_checklist.html`

Total checklist items: **42**

## Phase 0: Prerequisites & accounts setup (Before you write code)

1. **Read the OpenEnv GitHub repo completely**
   - Go through `meta-pytorch/OpenEnv`, read the `README`, `envs/README.md`, and the existing `CodingEnv` or `GitEnv` example so you understand what a complete environment looks like.
2. **Install openenv CLI locally**
   - Run `pip install openenv` and verify it installs without error. This CLI will be used to validate your submission at the end — you need it working early.
3. **Create a Hugging Face account**
   - Sign up at `huggingface.co` if you don't have one. You'll need to create a Space and push a Docker image there for final deployment.
4. **Create a GitHub repository**
   - Make a new public repo named `dataforge-openenv`. This is where all your code lives and what you'll submit. Add a `.gitignore` for Python.
5. **Set up your local Python environment**
   - Create a virtual environment with Python 3.11. Install your core dependencies: `openenv`, `fastapi`, `uvicorn`, `duckdb`, `pandas`, `openai`, `pydantic`. Pin versions in `requirements.txt` immediately.
6. **Verify Docker is installed and working**
   - Run `docker build` and `docker run` on a hello-world container. If Docker isn't installed, do that now — you'll need it for the final containerization step.

## Phase 1: Project structure & scaffolding (Day 1)

1. **Create the full folder structure**
   - Create all directories: `environments/sql_env/`, `environments/cleaning_env/`, `server/`, `baseline/`, and root-level files. Don't write code yet — just create empty files with correct names so the structure is right from the start.
2. **Create the openenv.yaml manifest file (draft)**
   - Write the yaml with metadata: name, version, description, both environment definitions (sql and cleaning), their action/observation space names, `reward_range [0.0, 1.0]`, and the 6 task IDs. This file evolves but having a draft forces you to commit to names early.
3. **Create requirements.txt with pinned versions**
   - List every dependency with exact versions. Judges will run `docker build` — if versions float, your build may break. Pin `openenv`, `fastapi`, `uvicorn`, `duckdb`, `pandas`, `openai`, `pydantic`, and any other library you add.
4. **Create a minimal README.md skeleton**
   - Add section headings now: Environment Description, Action Space, Observation Space, Tasks, Setup Instructions, Baseline Scores. Leave them empty for now but having the structure means you fill it in as you go, not in a panic at the end.

## Phase 2: SQL environment — models & database (Day 1-2)

1. **Design and write SQLAction model**
   - In `sql_env/models.py`, define `SQLAction` with a single field: `query (str)`. This is the only thing an agent can do — submit a SQL string. Keep it minimal and typed with Pydantic/dataclass as OpenEnv requires.
2. **Design and write SQLObservation model**
   - Define `SQLObservation` with: `success (bool)`, `error (Optional[str])`, `result_preview (Optional list of rows)`, `execution_time (Optional float)`, `plan_summary (Optional str)`, `reward (float)`, `done (bool)`. Every field serves a purpose — don't add fields you won't populate.
3. **Design and write SQLState model**
   - Define `SQLState` with: `episode_id (str)`, `task_id (str)`, `step_count (int)`, `schema_context (str — the DDL)`, `broken_query (str — what the agent sees)`, `target_hash (Optional str)`, `baseline_cost (Optional float for optimization task)`.
4. **Set up DuckDB and seed the database**
   - In `sql_env/db/`, create a seed SQL script that creates 3-4 tables (`orders`, `customers`, `products`, `line items`) with realistic column names and populates them with ~500-1000 rows of fake but plausible data. Run this once at environment startup.
5. **Create 10 broken queries for Task 1 (syntax)**
   - Write 10 valid SQL queries first, then inject one syntax error into each: a missing comma, a misspelled keyword like `SELCT`, an unclosed parenthesis, a missing `FROM` clause. Store them as a list of dicts with the broken query and the correct query.
6. **Create 8 broken queries for Task 2 (logic)**
   - Write queries that run without error but return wrong results: wrong JOIN type (`LEFT` vs `INNER`), off-by-one in `WHERE` filter, wrong column in `GROUP BY`, missing `DISTINCT`. Store the correct result hash alongside each.
7. **Create 6 slow queries for Task 3 (optimization)**
   - Write correct but deliberately inefficient queries: full table scans that could use filtering, Cartesian products, redundant nested subqueries, repeated aggregations. Record the baseline execution time from `EXPLAIN ANALYZE` for each.

## Phase 3: SQL environment — logic & graders (Day 2-3)

1. **Write the SQLEnvironment class with reset()**
   - In `sql_env/env.py`, implement `reset(task_id)`. It should: pick a random task from the task list for that task_id, open a DuckDB connection, load the seed data, return an initial `SQLObservation` showing the agent the schema and broken query.
2. **Write step() for Task 1 — syntax grader**
   - When the agent submits a query: try to execute it in DuckDB. If it parses and runs: `reward = 1.0`, `done = True`. If it fails: `reward = 0.0`, return the DuckDB error message in the observation so the agent can self-correct. Increment `step_count`.
3. **Write step() for Task 2 — logic grader**
   - Execute the agent's query and the gold query. Hash the sorted result sets. Exact match = `1.0`. For partial credit: compute row-level Jaccard similarity (intersection / union of rows). Return `result_preview` (first 5 rows) so agent can see what it got.
4. **Write step() for Task 3 — optimization grader**
   - Run `EXPLAIN ANALYZE` on both the baseline query and the agent's query. Extract `total_time` from DuckDB's JSON output. `reward = min(1.0, baseline_time / agent_time)`. If agent's query changes the result set, apply `-0.2` penalty and explain in `error` field.
5. **Write state() method**
   - Return the current `SQLState` object — immutable snapshot of `episode_id`, `task_id`, `step_count`, `schema_context`, and `baseline_cost`. This is a read-only view; calling it should not change anything.
6. **Add episode termination logic**
   - Decide and implement: max steps per episode (suggested: 10 for syntax, 15 for logic, 20 for optimization). When `step_count` hits max, set `done = True` and return final reward. This prevents infinite loops.

## Phase 4: Data cleaning environment — models & data (Day 3-4)

1. **Design and write CleanAction model**
   - Define `CleanAction` with: `action_type (Literal of impute, drop_duplicates, cast_type, standardize_format, split_column, done)`, `column (Optional str)`, `parameters (Optional Dict)`. The `done` action ends the episode — agent signals it's finished cleaning.
2. **Design and write CleanObservation model**
   - Define `CleanObservation` with: `success (bool)`, `error (Optional str)`, `rows_affected (int)`, `schema_diff (Dict — nulls remaining, type mismatches, duplicates count)`, `columns_clean (List[str])`, `reward (float)`, `done (bool)`.
3. **Design and write CleanState model**
   - Define `CleanState` with: `episode_id`, `task_id`, `step_count`, `columns_correct (list of fully cleaned columns)`, `total_columns (int)`, `gold_schema (Dict describing the target schema)`.
4. **Create dirty dataset for Task 4 (nulls & types)**
   - Generate a 500-row CSV of customer data with: null values represented as `'N/A'`, `'-'`, `''`, and `None` mixed together; an age column stored as strings like `'34 years'`; a salary column with `$` signs and commas; phone stored as integers. Keep a clean gold version.
5. **Create dirty dataset for Task 5 (dedup & format)**
   - Take the clean dataset and: duplicate 20% of rows with minor variations (extra space, different capitalization); mix 3 date formats in the same column (`DD/MM/YYYY`, `Mon DD YYYY`, Unix timestamp); mix phone formats. Keep gold version.
6. **Create dirty dataset for Task 6 (schema normalization)**
   - Create a denormalized dataset with: a `full_name` column that should be `first_name + last_name`; an `address` column combining city, state, and zip; UTF-8/latin-1 encoding artifacts like `Ã©` instead of `é`. Keep gold version with correct schema.

## Phase 5: Data cleaning environment — logic & graders (Day 4-5)

1. **Write the CleaningEnvironment class with reset()**
   - Load the appropriate dirty CSV for the `task_id`. Compute baseline quality metrics (null count per column, duplicate count, format violation count) and store them. Return initial `CleanObservation` with `schema_diff` showing current quality vs target.
2. **Implement each action type as a pandas operation**
   - Map each `action_type` to a pandas operation: `impute -> fillna()`, `drop_duplicates -> drop_duplicates()`, `cast_type -> astype()`, `standardize_format -> apply() with a regex`, `split_column -> str.split()`. Wrap each in `try/except` and return error in observation if it fails.
3. **Write step() grader for Task 4 — null/type scoring**
   - After each action, recompute: `null_score = 1 - (remaining_nulls / original_nulls)` per column, `type_score = columns with correct dtype / total columns`. Per-step reward = delta improvement from last step. Final reward = mean score across all columns when `done` action is called.
4. **Write step() grader for Task 5 — dedup/format scoring**
   - After each action: `dedup_score = 1 - (remaining_dupes / original_dupes)`, `format_score = fraction of rows matching target regex per column`. Per-step reward = weighted delta. Final reward = `0.5 x dedup_score + 0.5 x format_score`.
5. **Write step() grader for Task 6 — schema scoring**
   - After each action, compare current dataframe columns to gold schema. For each column: check name matches, dtype matches, and values match (using `pd.Series.equals` for exact, fuzzy for strings). reward per step = `newly_correct_columns / total_columns`.
6. **Add penalty for destructive actions**
   - If an action drops more than 30% of rows (too aggressive), apply a `-0.1` penalty and warn in the observation `error` field. This discourages agents from deleting their way to a clean dataset.

## Phase 6: FastAPI server (Day 5)

1. **Write the main FastAPI app in server/app.py**
   - Create three endpoints: `POST /reset` (takes `env_id` and `task_id`, returns initial observation), `POST /step` (takes `episode_id` and action dict, returns new observation), `GET /state` (takes `episode_id`, returns current state). Store active environment instances in a dict keyed by `episode_id`.
2. **Add request validation and error handling**
   - Every endpoint should validate input types and return clear HTTP 400 errors for bad input. If `episode_id` is not found, return 404. If an action is malformed, return the error in the observation rather than crashing the server.
3. **Add GET /health endpoint**
   - A simple endpoint that returns `{status: ok}`. This is required by the Dockerfile `HEALTHCHECK` instruction and also lets judges quickly verify your Space is alive.
4. **Test all endpoints manually with curl or Postman**
   - Before moving on, manually call `reset`, `step`, and `state` for both environments and all 6 tasks. Confirm observations are correct, rewards are sensible, and `done` triggers at the right time.

## Phase 7: Baseline agent script (Day 6)

1. **Write the baseline prompt builder**
   - In `baseline/agent.py`, write a function that takes an observation and formats it into a clear text prompt for the LLM. For SQL: include the schema, the broken query, the error message. For cleaning: include the `schema_diff` and column list. Be explicit about what action format the model should return.
2. **Write the action parser**
   - Write a function that takes GPT-4o's text response and extracts either a SQL query string (for SQL env) or a structured `CleanAction` (for cleaning env). Handle cases where the model returns markdown code blocks, extra explanation, or malformed output gracefully.
3. **Write the main run_task loop**
   - Call `reset()`, then loop: build prompt -> call OpenAI API -> parse action -> call step() -> accumulate reward -> check done. Log each step's reward to console. Return total reward for the episode.
4. **Run baseline on all 6 tasks and record scores**
   - Run the script with `OPENAI_API_KEY` set. Record the output scores for all 6 tasks. These exact numbers go into your README. Run it twice to confirm they're reproducible (scores may vary slightly for optimization task due to timing).

## Phase 8: Dockerfile & containerization (Day 6-7)

1. **Write the Dockerfile**
   - Use `python:3.11-slim` as base. Copy `requirements.txt` first and run `pip install` (this layer caches). Then copy the rest of the code. Set `WORKDIR` to `/app`. Expose port `7860` (Hugging Face default). Set `CMD` to start `uvicorn` on `0.0.0.0:7860`.
2. **Add HEALTHCHECK to Dockerfile**
   - Add the `HEALTHCHECK` instruction pointing to your `/health` endpoint. This is explicitly called out in the OpenEnv spec and judges will check for it.
3. **Run docker build locally and fix all errors**
   - Run `docker build -t dataforge-openenv .` and resolve every error. Common issues: missing dependencies in `requirements.txt`, relative import errors, files not being copied. Don't proceed until the build completes cleanly.
4. **Run docker run locally and verify endpoints**
   - Run the container and call all 3 endpoints from your local machine against the container. Confirm the server starts clean, responds correctly, and the health check passes.
5. **Run openenv validate against the running container**
   - With the container running, run `openenv validate` from your CLI. Fix any compliance errors it reports. This is a hard requirement — judges run this command.

## Phase 9: Hugging Face Space deployment (Day 7)

1. **Create a new Hugging Face Space**
   - Go to `huggingface.co/new-space`. Choose Docker as the SDK. Name it `dataforge-openenv`. Set it to public. Add the `openenv` tag in the Space metadata — this is required for hub integration.
2. **Push your Docker image to the Space**
   - Follow HF's Docker Space instructions to push your image. The Space will build and deploy automatically. Monitor the build logs for errors.
3. **Verify the Space is live and responding**
   - Once deployed, call your Space's public URL at `/health`, `/reset`, and `/step`. Confirm the responses match what you get locally. If anything is broken, debug via the HF Space logs.

## Phase 10: README & final submission (Day 7-8)

1. **Write the Environment Description section**
   - Explain what DataForge OpenEnv is, why you chose SQL debugging and data cleaning, and who would use this environment. 2-3 paragraphs. Make a compelling case for real-world utility — this is 30% of your score.
2. **Document Action and Observation spaces**
   - For each environment, write a clear table showing every field in the Action and Observation models with its type, whether it's optional, and what it represents. Judges read this to understand your design choices.
3. **Document all 6 tasks with difficulty ratings**
   - For each task: write the objective in one sentence, explain what the agent sees, explain what the grader checks, and state the difficulty (easy/medium/hard). Include the grader formula (e.g. `reward = baseline_time / agent_time`).
4. **Write setup instructions**
   - Cover: cloning the repo, installing dependencies, running locally with `uvicorn`, running with Docker, running the baseline script with `OPENAI_API_KEY`, and running `openenv validate`. Someone who has never seen your code should be able to run it in 10 minutes.
5. **Add baseline scores table**
   - Add a table showing the GPT-4o baseline scores for all 6 tasks. Include the model name, date run, and the exact command used. This proves reproducibility.
6. **Final check: run openenv validate one more time**
   - With the HF Space live, point `openenv validate` at the Space URL and confirm it passes. Take a screenshot or copy the output — include it in your README or submission notes.
7. **Submit your GitHub repo and HF Space link**
   - Submit both links on the hackathon platform before the April 8 deadline. Double-check that the GitHub repo is public, the HF Space is public, and the baseline script is in the repo with instructions.
