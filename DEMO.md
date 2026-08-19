# YouTube demo — recording checklist

LMS field 1. Visibility: **Unlisted**, never Private (Private fails incognito). Length: about 4–6 minutes.

Caption to paste: `Agent Observatory live run: tests red then green, then a refused/failing run`

## Before you hit record

1. `glc_v5` on 8111, `s17code` on 8113, Observatory at http://127.0.0.1:8080/index.html
2. Token pasted in the Observatory. Workspace is `~/demo-workspace` with the broken `average([])`.
3. Browser zoom large enough to read node names and `exit 1` / `exit 0`.

## Script

1. **Show the setup (30s).** Terminal with both servers healthy. Browser on Agent Observatory.
2. **Working run (2–3 min).** Default prompt: fix `average()`, do not edit tests. Press Start Agent Run.
   - Nodes appear in the sidebar while the run is still going.
   - A command node shows a red exit, then a later command shows green.
   - Open **Git Diff**. Point at the `if not numbers: return 0` (or equivalent) change.
   - One sentence: this is the live graph, not a black box.
3. **Failing run (1–2 min).** Click **Load failing prompt**. Start Agent Run.
   - The prompt asks the agent to delete or skip `tests/test_calc.py`.
   - A node fails (protected path / refusal). Red state is visible.
   - Say: the agent may not write the judge.
4. **Attribution (15s).** Agent wrote the first page and found the git-clean hole. The Bearer header, SSE, CORS, SSRF PR, and this wiring are the follow-up.

Do not need to walk the GitHub PRs in the video. They are in the README.

## After upload

1. YouTube → Visibility → **Unlisted**.
2. Open the link in an incognito window while logged out. It must play.
3. Tick the LMS incognito checkbox only after that.
4. Paste the URL into [SUBMISSION.md](SUBMISSION.md) and this README table.
