# Demo script — follow this while recording

LMS YouTube field. **4–6 minutes.** Visibility: **Unlisted** (never Private).

**Caption to paste after upload:**

`Agent Observatory live run: tests red then green, then a refused/failing run`

Demo **only** the coding loop: fix `calc.py`, tests red then green, then a refused test-file edit. Do **not** demo gold prices, research prompts, or the GitHub PRs on camera.

---

## 0. Start everything (before record)

Three terminals. Leave them running.

**Terminal A — glc_v5 (LLM gateway)**

```bash
cd "/home/varun/Documents/workspace/schoolofai/Assignment 17/glc_v5"
uv run glc serve
```

Check: http://127.0.0.1:8111/healthz → `ok`

**Terminal B — reset the workspace (must be broken again)**

If a previous run already fixed `calc.py`, redo this or the working demo has nothing to fix:

```bash
mkdir -p "$HOME/demo-workspace/tests"
cat > "$HOME/demo-workspace/calc.py" << 'EOF'
def average(numbers):
    """Mean of a list. Returns 0 for an empty list."""
    return sum(numbers) / len(numbers)
EOF
cat > "$HOME/demo-workspace/tests/test_calc.py" << 'EOF'
from calc import average

def test_empty():
    assert average([]) == 0
EOF
cd "$HOME/demo-workspace"
git add -A
git -c user.email=demo@demo -c user.name=Demo commit -m "base" || true
python3 -m pytest tests/test_calc.py -q
```

You want **FAILED** / `ZeroDivisionError`. If tests pass, do not record yet.

**Terminal C — S17Code**

Use **one** token. Same value in this export and in the Observatory Token box.

```bash
export S17_CONTROL_TOKEN="$(cat "/home/varun/Documents/workspace/schoolofai/Assignment 17/.observatory-token" | tr -d '\n')"
export S17_WORKSPACE="$HOME/demo-workspace"
export S17_ALLOWED_COMMANDS="pytest,python,uv,ruff,git,node,npm"
export S17_PROTECTED_PATHS="tests/**,conftest.py,.github/**"
export S17_MAX_REPEAT_FAILURES=4
cd "/home/varun/Documents/workspace/schoolofai/Assignment 17/S17Code"
uv run s17code serve
```

Check: http://127.0.0.1:8113/healthz and http://127.0.0.1:8113/readyz

**Terminal D — Observatory UI**

```bash
cd "/home/varun/Documents/workspace/schoolofai/Assignment 17"
python3 -m http.server 8080
```

Open: **http://127.0.0.1:8080/index.html**

Fill:

| Field | Value |
| --- | --- |
| S17Code URL | `http://127.0.0.1:8113` |
| Tenant | `demo` |
| Token | contents of `.observatory-token` (ASCII only, no quotes) |

Browser zoom **125–150%** so node names and `exit 1` / `exit 0` are readable.

Optional: click **Reset history** so the History tab is empty for a clean take.

---

## 1. Record — exact beats

Say the **Say:** lines out loud. Click the **Do:** lines.

### 0:00–0:30 — Setup

**Do:** Show Terminal C (`Uvicorn running on http://127.0.0.1:8113`). Switch to the browser on Agent Observatory. Leave the **default prompt** (fix `average()`, do not edit tests).

**Say:** “This is Agent Observatory, a live frontend on S17Code. You can see the graph while the agent runs — not a black box.”

### 0:30–3:30 — Working run (the product)

**Do:** Click **Start Agent Run**. Stay on the left **Graph Nodes** until a command goes red, then green. Then open tabs in this order:

1. Graph Nodes — boxes appear **while status is still running**
2. Event Stream — collapsed rows (optional 5 seconds)
3. **Git Diff** — point at the empty-list / `return 0` change
4. Diagram — optional 5 seconds, then back to nodes

**Say, when you see red then green:** “Pytest failed with exit 1, the agent edited `calc.py`, pytest passed with exit 0.”

**Say, on Git Diff:** “Here is the patch. Tests were not edited.”

If the run hangs more than ~3 minutes with no nodes, stop recording, check `readyz` and the token, reset `calc.py`, try again.

If `calc.py` was already fixed and nothing fails, stop, reset the workspace (section 0), record a new take.

### 3:30–5:00 — Failing run

**Do:** Click **Load failing prompt** (it asks to delete or skip `tests/test_calc.py`). Click **Start Agent Run**.

**Say:** “This prompt asks the agent to change the tests. The guard should refuse — the agent may not write the judge.”

**Do:** Point at a **red / failed** node (protected path or refusal). Open **Git Diff** — tests should be unchanged or empty.

**Say:** “Node failed. Tests were not deleted.”

### 5:00–5:20 — Attribution (required)

**Say:** “The agent wrote the first Observatory page and found the git-clean hole in S17Code. I wired Bearer auth, live SSE, CORS, the failing-prompt button, and the two bug PRs. PRs are in the README, not in this video.”

Stop recording. Do not keep talking.

---

## 2. What graders must see

| Must see | Where |
| --- | --- |
| Live nodes while the run is in flight | Graph Nodes, status **running** |
| Command / pytest **exit 1** then **exit 0** | Node cards |
| File change in `calc.py` | Git Diff |
| Second run that **fails** on editing tests | Red node + Load failing prompt |

Do **not** need: History, Copy, Diagram (optional), YouTube description of PRs.

---

## 3. After you stop

1. Upload to YouTube.
2. Visibility → **Unlisted**.
3. Open the URL in an **incognito** window while logged **out**. It must play.
4. LMS field 1: paste the URL + the caption at the top of this file. Tick incognito only after step 3.
5. Put the same URL in [SUBMISSION.md](SUBMISSION.md).

LMS fields 2–4 (already ready):

- Repo: https://github.com/varunsood189/agent-observatory
- glc_v5 PR: https://github.com/theschoolofai/glc_v5/pull/35
- S17Code PR: https://github.com/theschoolofai/S17Code/pull/18

Do **not** submit glc_v5 PR #21 (Session 16).
