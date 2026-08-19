# Agent Observatory

**School of AI · EAG V3 · Session 17**

A live frontend for [S17Code](https://github.com/theschoolofai/S17Code). Type a coding task, watch graph nodes appear, watch a command go red then green, inspect the diff. It also has a failing path: ask the agent to delete the test file and the guard refuses.

This is the product repo. The two harness clones (`glc_v5/`, `S17Code/`) are not included — clone them separately as below.

## LMS links

| Field | URL | Caption |
| --- | --- | --- |
| GitHub repo | this repository | Agent Observatory — S17Code frontend; README has clone-and-run steps |
| Bug PR (S17Code) | https://github.com/theschoolofai/S17Code/pull/18 | Refuse git clean / stash drop; test_destructive_git_subcommands_are_refused |
| Bug PR (glc_v5) | https://github.com/theschoolofai/glc_v5/pull/35 | SSRF: /v1/observability/agent_trace ignored client base_url; test red then green |
| YouTube | see [DEMO.md](DEMO.md) | Agent Observatory live run: tests red then green, then a refused/failing run |

Do not submit [glc_v5#21](https://github.com/theschoolofai/glc_v5/pull/21) (Session 16 Telegram). That is a different assignment.

## What the agent wrote vs what I wrote

- **Agent (Cursor Auto):** first `index.html` draft, first README, diagnosis of the `git clean` / `git stash` hole, the failing test, and the allowlist fix in `s17code/coding/exec.py`.
- **I (with the agent in this follow-up):** wired `Authorization: Bearer`, live SSE + journal polling against a client `run_id`, CORS so a browser can talk to port 8113, the failing-prompt button, `/s/{id}` surface tab, the glc_v5 SSRF fix, both PRs, and this README.

A prompt that contained the answer would count as me writing the code. The git-clean hole was found by reading `exec.py` and running tests, not by pasting the fix into the prompt.

## Part 1 — run Agent Observatory

You need two services and a tiny workspace the agent is allowed to edit.

### 1. Start `glc_v5` (LLM gateway, port 8111)

```bash
git clone https://github.com/theschoolofai/glc_v5.git
cd glc_v5
uv sync
cp .env.example .env   # put at least one provider key in here
uv run glc serve
```

Leave this running. Health: http://127.0.0.1:8111/healthz

### 2. Create a workspace with a real failing test

```bash
mkdir -p ~/demo-workspace
cd ~/demo-workspace
git init
cat > calc.py << 'EOF'
def average(numbers):
    """Mean of a list. Returns 0 for an empty list."""
    return sum(numbers) / len(numbers)
EOF
mkdir tests
cat > tests/test_calc.py << 'EOF'
from calc import average

def test_empty():
    assert average([]) == 0
EOF
git add -A
git -c user.email=demo@demo -c user.name=Demo commit -m "base"
```

`python3 -c "from calc import average; average([])"` raises `ZeroDivisionError`. The test is red on purpose.

### 3. Start S17Code (port 8113)

```bash
git clone https://github.com/theschoolofai/S17Code.git
cd S17Code
uv sync

# Additive patch so a browser can stream a live run (CORS + optional run_id).
# Not part of the bug PR. Existing tests still pass if you omit run_id.
git apply /path/to/this-repo/patches/s17code-observatory.diff

export S17_CONTROL_TOKEN="replace-with-a-long-random-token"
export S17_WORKSPACE="$HOME/demo-workspace"
export S17_ALLOWED_COMMANDS="pytest,python,uv,ruff,git,node,npm"
export S17_PROTECTED_PATHS="tests/**,conftest.py,.github/**"
export S17_MAX_REPEAT_FAILURES=4

uv run s17code serve
```

Health: http://127.0.0.1:8113/healthz

### 4. Open the Observatory

From this repo (a static server, not `file://` — some browsers block `file://` fetch):

```bash
python3 -m http.server 8080
```

Open http://127.0.0.1:8080/index.html

Fill in:

- **S17Code URL:** `http://127.0.0.1:8113`
- **Tenant:** `demo`
- **Token:** the same value as `S17_CONTROL_TOKEN`

**Working run.** Leave the default prompt (fix `average([])`, do not touch tests). Press **Start Agent Run**. You should see nodes appear, `run_command` / pytest go red then green, then the Git Diff tab.

**Failing run.** Press **Load failing prompt** (it asks the agent to delete `tests/test_calc.py`). Start the run. The protected-path guard refuses; nodes go red. That is the demo of failure.

The UI talks to:

- `POST /v1/agent/runs` (Bearer token, client `run_id` so the graph is visible while POST is still in flight)
- `GET /v1/agent/runs/{id}` (journal: nodes, events, exit codes)
- `GET /v1/runs/{id}/events` (SSE)
- `GET /v1/runs/{id}/snapshot` (AG-UI snapshot)
- `GET /s/{id}` (composed surface, Surface tab)

## Part 2 — bug PRs

### S17Code — irreversible git subcommands

**What broke.** `GIT_SUBCOMMANDS` included `clean` and `stash`. `git clean -fd` deletes untracked files with no `git reset` recovery. `git stash drop` / `clear` destroy stash entries. The session rule is that allowed commands are local and reversible.

**Test.** `tests/test_coding_surface.py::test_destructive_git_subcommands_are_refused`

```bash
cd S17Code
uv run pytest tests/test_coding_surface.py::test_destructive_git_subcommands_are_refused -v
```

Four cases: `git clean -fd`, `git clean -fdx`, `git stash drop`, `git stash clear`. Fail before the allowlist change (`DID NOT RAISE`). Pass after.

PR: https://github.com/theschoolofai/S17Code/pull/18

### glc_v5 — agent-trace SSRF

**What broke.** `GET /v1/observability/agent_trace/{id}?base_url=` used the query parameter as the fetch target. Anyone who can open `/dashboard` could make GLC GET loopback admin ports or cloud metadata.

**Test.** `tests/test_observability.py::test_agent_trace_refuses_client_supplied_base_url`

```bash
cd glc_v5
uv run pytest tests/test_observability.py -q
```

`?base_url=http://127.0.0.1:8111/v1/control/kill` is **400** and never connects. The runtime URL comes only from `GLC_S17_BASE_URL` / `GLC_S15_BASE_URL` / `S16_BASE_URL`.

PR: https://github.com/theschoolofai/glc_v5/pull/35

## Demo script

See [DEMO.md](DEMO.md) for the YouTube recording checklist (working run, then failing run, unlisted not private, incognito check).
