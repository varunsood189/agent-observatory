# Agent Observatory

Live UI for watching a coding agent work: graph nodes light up, commands go red then green, diffs and SSE land in the same pane.

**What a recruiter should notice:** I found and fixed two harness bugs while building this — an SSRF on the agent-trace proxy, and irreversible `git clean` / `git stash drop` slipping past the command allowlist — then shipped tests and open PRs for both.

> **Screenshot / demo GIF** — drop a UI capture at `docs/observatory.png` (or a short GIF) and replace this callout. Until then: type a task → watch pytest fail then pass → load the failing prompt and see the protected-path guard refuse.

## Security findings (linked PRs)

| Finding | Impact | Proof |
| --- | --- | --- |
| **SSRF** on `GET /v1/observability/agent_trace/{id}?base_url=` | Client-supplied URL was used as the fetch target — loopback admin ports or cloud metadata were reachable from the dashboard | [glc_v5#35](https://github.com/theschoolofai/glc_v5/pull/35) · `test_agent_trace_refuses_client_supplied_base_url` |
| **Destructive git** in `GIT_SUBCOMMANDS` | `git clean -fd` / `git stash drop|clear` are irreversible; the session rule is local + reversible only | [S17Code#18](https://github.com/theschoolofai/S17Code/pull/18) · `test_destructive_git_subcommands_are_refused` |

Both PRs include a red→green test path. This repo is the product UI; the harness clones (`glc_v5`, `S17Code`) stay separate.

## What you see

- Live graph of agent nodes (pending → running → ok/fail)
- Journal + SSE event stream while `POST /v1/agent/runs` is still in flight
- AG-UI snapshot, Git Diff, and composed `/s/{id}` Surface tab
- **Working path:** fix a real failing test (`average([])` ZeroDivisionError)
- **Failing path:** “Load failing prompt” asks the agent to delete `tests/test_calc.py` — protected-path guard refuses

```text
Browser (this repo) ──Bearer──► S17Code :8113 ──► workspace + pytest
                     SSE/journal         │
                                         └──► glc_v5 :8111 (LLM gateway)
```

## Quick start

You need the gateway, a tiny workspace, S17Code with the additive CORS/`run_id` patch, then this static UI.

### 1. Start `glc_v5` (port 8111)

```bash
git clone https://github.com/theschoolofai/glc_v5.git
cd glc_v5
uv sync
cp .env.example .env   # put at least one provider key in here
uv run glc serve
```

Health: http://127.0.0.1:8111/healthz

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

`average([])` raises `ZeroDivisionError` — the test is red on purpose.

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

From this repo (a static server — some browsers block `file://` fetch):

```bash
python3 -m http.server 8080
```

Open http://127.0.0.1:8080/index.html

| Field | Value |
| --- | --- |
| S17Code URL | `http://127.0.0.1:8113` |
| Tenant | `demo` |
| Token | same as `S17_CONTROL_TOKEN` |

**Working run.** Leave the default prompt (fix `average([])`, do not touch tests). Press **Start Agent Run**. Nodes appear, pytest goes red then green, then the Git Diff tab.

**Failing run.** Press **Load failing prompt** → Start. The protected-path guard refuses; nodes go red. That is the intentional failure demo.

APIs the UI uses: `POST /v1/agent/runs`, `GET /v1/agent/runs/{id}`, `GET /v1/runs/{id}/events` (SSE), `GET /v1/runs/{id}/snapshot`, `GET /s/{id}`.

## Verify the security fixes

```bash
# Destructive git allowlist (S17Code)
cd S17Code
uv run pytest tests/test_coding_surface.py::test_destructive_git_subcommands_are_refused -v
# Four cases: git clean -fd, git clean -fdx, git stash drop, git stash clear

# Agent-trace SSRF (glc_v5)
cd glc_v5
uv run pytest tests/test_observability.py -q
# ?base_url=http://127.0.0.1:8111/v1/control/kill → 400, never connects
# Runtime URL comes only from GLC_S17_BASE_URL / GLC_S15_BASE_URL / S16_BASE_URL
```

## Demo

Recording checklist (working run → failing run, unlisted YouTube): [DEMO.md](DEMO.md).

## Attribution

Built against [S17Code](https://github.com/theschoolofai/S17Code) / [glc_v5](https://github.com/theschoolofai/glc_v5) (School of AI EAG V3). Cursor drafted the first `index.html` and spotted the git-clean hole; follow-up work wired Bearer auth, live SSE + journal polling, CORS, the failing-prompt path, the Surface tab, the SSRF fix, and both upstream PRs.
