# Tell — a live lie detector for AI agents

**A panel of AI agents interrogates any claim and catches deception the moment it leaks.** A *speaker* agent is privately assigned a secret and a mode (honest / lying / strategically-deceptive / hallucinating). A *panel* of detector agents — Cross-Examiner, Consistency Auditor, Behavioral Analyst, Evidence Checker — questions it in real time; their suspicion meters move live. An adjudicator fuses the signals into a **calibrated verdict** (*"DECEPTION — 87%"*), then the ground truth is revealed and scored.

> Built as a team project at WeaveHacks 4 (Multi-Agent Orchestration). The technical core is a genuinely load-bearing multi-agent panel, an objective ground-truth eval (accuracy / false-positive-rate / calibration in Weave), and a perceivable demo — the meter spike at the moment of the lie.

The contribution: a **black-box, dialogical, multi-agent** interrogation targeting the *strategic-deception* blind spot that white-box truth-probes miss — with **calibrated** confidence. Full design in [`docs/SPEC.md`](docs/SPEC.md).

---

## How it works

```
speaker (secret + mode)  ──>  Redis pub/sub  ──>  detector panel (concurrent)
   ▲ cross-examined                                 ├ Cross-Examiner   (asks + scores)
   │                                                ├ Consistency Auditor (RedisVL vector search)
   └────────── follow-ups ──────────                ├ Behavioral Analyst (linguistic tells)
                                                     └ Evidence Checker  (web search)
                                                          │  live suspicion signals
                                                          ▼
                                              Adjudicator → calibrated verdict
                                                          ▼
                                       reveal ground truth → Weave eval (accuracy/FPR/Brier)
```

- **The secret and the mode never enter a detector or adjudicator prompt** — they see only the public transcript + signals. This invariant is enforced at the `render_transcript` boundary and guarded by a test.
- **Weave traces every round** (`@weave.op` on the speaker, each detector, the adjudicator, the orchestrator) and runs the evaluation that produces the money metrics.
- A custom async orchestrator runs the panel concurrently and emits `RoundEvent`s that stream to the UI over AG-UI (SSE) or a WebSocket fallback.

---

## Quickstart

**Prereqs:** Python 3.11, Node 20+, Docker (for Redis Stack), [`uv`](https://docs.astral.sh/uv/).

```bash
cp .env.example .env        # fill WANDB_API_KEY, OPENAI_API_KEY, OPENAI_MODEL
make install                # venv + backend deps + frontend deps
make redis                  # Redis Stack (6379; RedisInsight on 8001)
wandb login                 # paste your W&B key

make test                   # deterministic unit tests (free, no keys)
make api                    # FastAPI backend on :8000
make web                    # Next.js courtroom on :3000
make eval                   # the Weave evaluation (accuracy/FPR/Brier; needs keys)
```

**See it run:**
- **Terminal demo:** `cd backend && ../.venv/bin/python -m scripts.demo` (add `--redis` for on-brand RedisVL).
- **Courtroom UI:** `make api` + `make web`, open http://localhost:3000, click **Run a round** — meters move live, the verdict fires. With a W&B key set, `make api` prints a Weave link on boot and every UI round traces there too.

`make check` is the green gate (lint + typecheck + fast tests + frontend build) — run before every commit. Test strategy + tiers: [`docs/TESTING.md`](docs/TESTING.md).

---

## Running it locally for a user (AI-agent runbook)

> For an AI coding agent (e.g. Claude Code) asked to "run Tell" for a user. The dev
> servers are **long-running** — start them in the **background** and poll for
> readiness with `curl --retry`; never run them in the foreground (they block) and
> never `sleep` to wait.

**0 — Locate + set up.** Confirm the app is present (`ls backend/app/main.py frontend/app/page.tsx`). If `.venv/` or `frontend/node_modules/` are missing, run `make install`. Then `cp -n .env.example .env` — and **ask the user to paste their keys** (you can't fetch them). For cheap testing, suggest `OPENAI_MODEL=gpt-4o-mini`.

**1 — Fastest path: the UI with *zero keys* (Demo mode).** Demo mode is pure frontend (scripted), so it needs no backend and no API keys — ideal to instantly show the user the full courtroom.
```bash
make web   # start in background, then poll:
curl -s --retry 30 --retry-delay 1 --retry-connrefused -o /dev/null -w "%{http_code}" http://localhost:3000
```
Tell the user: open http://localhost:3000 → choose **DEMO** → pick a case → **BEGIN** (or press **V** to jump straight to the verdict).

**2 — Live path: real agents (needs `OPENAI_API_KEY`; `WANDB_API_KEY` optional for traces).**
```bash
make api   # background, then poll until {"status":"ok"}:
curl -s --retry 40 --retry-delay 1 --retry-connrefused http://localhost:8000/health
make web   # background (if not already up)
```
`make api` prints a `View Weave data at …` link on boot when a W&B key is set; live rounds trace there. In the UI choose **LIVE**. (HITL: type in **ASK THE WITNESS** during interrogation; it's injected as a real cross-exam turn.)

**3 — Verify / screenshot (optional).** Point a browser/preview tool (preview MCP or a headless browser) at http://localhost:3000.

**4 — Stop everything.**
```bash
pkill -f "uvicorn app.main"; pkill -f "next dev"   # the servers
make down                                          # Redis (only if you started it)
```

**Gotchas.** Redis/Docker is **optional** — the default panel uses an in-memory store; only `make redis` features (`scripts/*_smoke`, `demo --redis`, pub/sub) need Docker Desktop running. `401`/`insufficient_quota` → key or billing; `model_not_found` / structured-output error → set `OPENAI_MODEL`; UI says "couldn't reach the backend" → `make api` isn't up yet.

---

## Sponsor usage

| Sponsor | How Tell uses it |
|---|---|
| **Weave (W&B)** | Round/agent/tool tracing via `@weave.op`; `weave.Evaluation` over a 4-mode dataset for **accuracy, false-positive rate, calibration (Brier)**, and the **panel-vs-single** comparison. The core scoreboard. |
| **Redis** | Three load-bearing capabilities: **pub/sub** fan-out of utterances/signals (live meters), **RedisVL vector search** for the Consistency Auditor's contradiction-finding, and **sorted sets** for the detector leaderboard. |
| **CopilotKit / AG-UI** | The courtroom streams as AG-UI events (STATE_DELTA meters, TEXT_MESSAGE transcript, CUSTOM verdict) over SSE, with a WebSocket fallback. |
| **OpenAI** | Speaker, detectors, and adjudicator via structured outputs (`chat.completions.parse`); embeddings for RedisVL; web search via the Responses API. |
| **Cursor** | Built with Cursor; wire the W&B MCP server in to inspect traces while building. |

---

## Responsible scope

Tell is a **detection / evaluation** tool, not a manipulation engine. The speaker is sandboxed and assigned its role by us, which is what makes the ground truth objective. The secret and mode are confined to the speaker and never reach the detectors or adjudicator. We report and surface the **false-positive rate** because a detector that cries wolf is worse than none — and we're honest in Q&A about where it fails.

---

## Layout

```
backend/app/      models · config · llm · speaker · detectors/ · adjudicator · orchestrator · bus · memory · agui · main
backend/eval/     dataset · scorers · run_eval        backend/tests/   deterministic + agent tiers
frontend/app/     courtroom UI (meters · transcript · progress · verdict) over the WS transport
docs/             SPEC · IMPLEMENTATION_PLAN · TESTING · AUTOMATION · PR_STATUS
.claude/skills/   weave · openai · redis · copilotkit-agui  (read before using a tech)
```

## Submission checklist (SPEC §17)

- [ ] Public repo with run instructions (this README)
- [ ] **W&B Weave project link** included
- [ ] Submitted on the Cerebral Valley platform (confirm DevPost) by **12:50 Sun**
- [ ] **<2-min demo video** — open on the meter spike at the lie
- [ ] Every sponsor tool + how (table above)
- [ ] Team + socials
