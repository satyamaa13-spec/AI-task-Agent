# Autonomous AI Task Worker (prototype)

Give it a natural-language office task; it operates a (sandboxed) browser and files, decides its own steps,
recovers from errors, asks for approval/clarification when needed, **verifies the outcome independently**, and
returns a summary with evidence.

Demo task (included sandbox):
> *Find the latest invoice from Globex Corp in the vendor portal, enter it as a bill in the ERP, and tell me when it's done.*

## Setup & run

```bash
pip install -r requirements.txt          # only dependency: anthropic
export ANTHROPIC_API_KEY=sk-ant-...
python run.py "Find the latest invoice from Globex Corp in the vendor portal, enter it as a bill in the ERP, and confirm once done."
```
Gemini instead of Claude: `pip install google-genai`, set `GEMINI_API_KEY`, add `--provider gemini` (optionally `--model <name>`).
Request budget (useful on free API tiers): `--max-llm-calls 9` hard-caps model requests; `--no-llm-verifier` skips the extra verifier call
(the read-after-write gate still applies). The UI has the same two options. The result shows how many model calls were used.
Flags: `--auto-approve` (skip the y/N prompt for gated actions), `--no-flaky` (disable the ERP fault), `--model`, `--max-steps`.
**Browser UI (recommended for the demo):** `python ui.py` opens http://127.0.0.1:8765 with a task box, live feed of the agent's
thoughts/actions, Approve/Deny buttons, the simulated ERP beside it, and the stored-data ground truth. Choose Gemini or Claude in the dropdown.
("Scripted replay" is a fixed script to preview the UI without a key; it is not AI. Do not present it as the agent.)

**Test data without editing code:** in the UI click "Add test data" (right panel) to add invoices through a form or load a 5-invoice
test pack (ambiguous "latest", over-limit amount, EUR, an id the ERP rejects). Added invoices persist in `sandbox/custom_invoices.json`.
The agent's browser is blocked from `/admin`, so it cannot edit its own test data.

Each run writes `runs/<timestamp>/{trace.jsonl, report.md, result.json}`. Run the sandbox alone: `python -m sandbox.server` (port 8000).
Harness tests (no API key, no network): `python -m unittest discover -s tests -t .`

Other tasks to try (zero code changes): *"Read the finance inbox note and do what it asks"*; *"Enter the latest Initech invoice"*;
*"Enter the latest Acme invoice"* (nonexistent vendor, so it should report that or ask rather than invent data).

## Architecture

```
run.py ─ starts sandbox, wires everything
 ├─ worker/agent.py   loop: LLM decides -> harness gates & executes tool -> observation -> repeat
 │                    + approval gate, verification gate, loop guards, context trimming, memory, tracing
 ├─ worker/tools.py   open_page, click_link, submit_form, list_dir, read_file, remember, ask_user (+ finish in agent)
 ├─ worker/browser.py HTTP+cookies+HTML parser -> text snapshot with numbered links/forms (no JS)
 ├─ worker/llm.py     AnthropicLLM (real) and ScriptedLLM (tests only)
 ├─ ui.py + ui/index.html  local web UI (same agent; WebIO answers approvals/questions via the browser)
 ├─ worker/io.py      CLI channel for questions/approvals/logs (swappable)
 └─ sandbox/server.py simulated company: vendor portal (login, unsorted invoices) + ERP (strict validation, flaky commit)
```
The task-specific knowledge lives only in the model's reasoning and in `run.py`'s environment description.
Nothing in `worker/` mentions invoices.

## Key design decisions

1. **One tool-use loop, no hand-written plan graph.** The model plans in-context and re-plans from observations. A fixed
   pipeline would be brittle and task-specific; a loop generalises.
2. **The harness enforces what the model shouldn't be trusted to.** Prompts ask for good behaviour; code guarantees it:
   - *Verification gate:* `finish("done")` is rejected if a write was sent and no successful read happened since. Then an
     **independent verifier call** (sees raw observations, not the agent's reasoning) must agree the evidence proves the outcome.
   - *"Error ≠ not executed":* a write that returns HTTP 500 still counts as a write. The sandbox ERP commits and then 500s;
     the agent must check state before retrying, otherwise it creates a duplicate (ERP also 409s duplicates).
   - *Approval gate:* generic risk rules (POST with amount over a limit, or action names like delete/pay/transfer) require user
     approval; denial returns an instruction to stop, not a puzzle to route around. Approvals are cached per action+amount so
     retries don't re-prompt.
   - *Loop guards:* identical call failing 3x or 5 consecutive failures injects a "change approach / ask user" nudge; step budget caps runaway runs.
3. **Working memory outside the transcript.** Old observations are trimmed to bound context; `remember` notes are re-injected
   into the system prompt every step, so key facts (source values) survive.
4. **Text browser instead of Playwright.** Fully deterministic, no dependencies, easy to debug. The `Browser` class is the only
   thing to swap; tools and agent stay unchanged.
5. **Sandbox is hostile on purpose.** Login redirect, unsorted list where the highest invoice number is *not* the newest,
   date/amount format mismatches, commit-then-500. A demo that can't fail proves nothing.
6. **Safety by construction:** host allow-list in the browser, workspace path confinement, fake credentials only.

## Models, APIs, components
Anthropic Messages API with tool use (default `claude-sonnet-5-5`, override via `WORKER_MODEL`/`--model`); optional Google Gemini adapter (`--provider gemini`, default `gemini-2.5-flash`, untested against the live API). Python 3 standard
library for everything else (`http.server`, `urllib`, `html.parser`, `unittest`). No agent framework, no pre-built agents.

## Testing: what is and isn't verified
`tests/test_harness.py` uses a **scripted LLM** to exercise the machinery against the real sandbox: login redirect, 422 format
error, commit-then-500, premature-finish rejection, approval gating/denial, verifier rejection, host/path/form-index errors.
These tests prove the *harness* is correct; they do **not** measure model intelligence. End-to-end quality must be judged by running
`run.py` with a real model (see demo). I could not run a live model in the build environment, so record your own demo run.

## Assumptions
Sandbox/fake systems only; one user; tasks are web/file-based; the portal credentials exist in the workspace; the verifier model
is the same model family as the worker (separate call, separate context).

## Known limitations
- No JavaScript, screenshots, file uploads/downloads, or PDFs; sites that need JS won't work with this `Browser`.
- Approval rules are heuristic (keyword/amount); no per-user policy or audit trail beyond `trace.jsonl`.
- "Read after write" is a coarse proxy for verification; the LLM verifier covers semantics but can itself be wrong.
- Verifier shares a model family with the worker (correlated blind spots).
- Single task at a time; no persistence of memory across runs; no cost/latency budgeting beyond step cap.
- Prompt-injection from page content is only partly mitigated (host allow-list + approval gate), not solved.

## What I'd build next
1. Playwright `Browser` (accessibility-tree snapshots + screenshots) behind the same interface.
2. Prompt-injection defence: mark tool output as untrusted data, taint-track values that originate from pages before they reach writes.
3. Deterministic post-conditions per task (declared by the user/ops team) in addition to the LLM verifier.
4. Idempotency keys and a transaction log for writes; durable cross-run memory; resume after crash.
5. Eval suite: dozens of generated sandbox scenarios with injected faults, scoring success rate, steps, and duplicate/unsafe-write rate across models.
6. Web UI for approvals/clarifications with the live trace and evidence screenshots.

## Suggested demo video (3-4 min)
Run the demo task live with a real key: show it (1) reading credentials and logging in after the redirect, (2) choosing INV-1042 by
*date* not number, (3) the approval prompt, (4) the 422 then the 500, (5) it re-reading the ERP and **not** duplicating, (6) the
verifier note and final report. Finish by printing the "ground truth in ERP database" line and running the unit tests.
