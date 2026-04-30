# hub-binary-gate-protocol-v2.md
# System: hub.otsbroker.com | Enforced AI Development Protocol
# Built: 2026-04-29 | Last updated: 2026-04-30 (8 gate upgrades — see changelog at bottom)
# Authority: This file governs ALL AI development sessions on Hub. It cannot be bypassed.

---

## RULE ZERO

Every session ends in exactly one state: DEPLOYED or BLOCKED.
There is no PARTIAL. There is no COMPLETE without a browser-verifiable change on hub.otsbroker.com.

---

## GATE SEQUENCE

G0 SCOPE_LOCK → G1 DESIGN_ALIGNMENT → G2 RISK_ANALYSIS → G3 IMPLEMENTATION → G4 CODE_QUALITY → G5 DEPLOY_VERIFY → G6 SESSION_SEAL

Every gate returns exactly: YES or NO
NO at any gate = session stops immediately. Rollback executes. Human is emailed. BLOCKED receipt is written.

Incremental path (minor changes only): G2 → G4 → G5 → G6

---

## ROLE MAP

| Role | Model | Allowed | Forbidden |
|---|---|---|---|
| Designer | GPT-5.5 → GPT-5.4 fallback | Propose scope | Gate own proposals, write code |
| Challenger | DeepSeek Reasoner | Challenge proposals, adversarial review | Propose features |
| Implementer | Claude Sonnet | Write code, deploy, provide evidence | Approve, validate, close gates |
| Gate Executor | GPT-4o-mini | Run all binary YES/NO gates | Propose, implement |
| Virtual JP | Gemini 2.5 Flash | Block if work contradicts project goals | Implement, propose |
| Virtual Michal | DeepSeek Reasoner (full) / deepseek-chat (incremental) | Block if quality, tests, or security fail | Propose, gate direction |

Hard rule: The model that produces an artifact cannot be the model that gates it.

Model cascade rule (designer/judge): GPT-5.5 is primary. If unavailable, fall back to GPT-5.4. If both unavailable, session hard stops. Never silently fall through to any other model.

Virtual JP gates: G0, G1, G6 (direction and outcome)
Virtual Michal gates: G2, G3, G4, G5 (implementation quality at every step)

---

## PRE-FLIGHT (runs before G0 — session blocked if any fail)

```bash
cd /home/otsadmin
bash /home/otsadmin/virtual/gate_v2/session_start_preflight.sh
```

If preflight fails: STOP. Do not proceed to G0. Email jurand@otsbroker.com.

---

## PROJECT MEMORY (loaded before every gate)

Full project memory is loaded once at session start:
- /home/otsadmin/.claude/projects/-home-otsadmin/memory/MEMORY.md — full text
- /home/otsadmin/CLAUDE.md — full text
- Last 5 receipts from /home/otsadmin/virtual/supervisors/receipts/ — full content

All gates receive this context. No gate runs blind.

```python
from gate_v2.run_gate import start_session
result = start_session()
SESSION_ID = result["session_id"]
CONTEXT = result["context"]
```

---

## PROMPT GATE (runs on every user prompt)

Every prompt is classified before any work begins:

```python
from gate_v2.run_gate import process_prompt
result = process_prompt(prompt=USER_PROMPT, session_id=SESSION_ID, changed_files=[])
# If result["overall_decision"] == "BLOCKED": stop immediately
# If result["route"] == "READ_ONLY": answer directly, no gates
# If result["route"] == "INCREMENTAL_GATE": G2+G4 only
# If result["route"] == "FULL_GATE": full G0→G6 sequence
```

---

## G0 — SCOPE_LOCK

Purpose: One user-visible change defined. DeepSeek challenges it. GPT-4o-mini gates.

Virtual JP reviews after DeepSeek challenge. JP block = session terminates.

Evidence required for YES:
- One-sentence scope: what changes in the browser, which page, which user role
- Exact files to change (max 5)
- Acceptance selector: exact text/element that will be present after deploy
- Rollback command
- DeepSeek has raised one concrete challenge and GPT-5.4 answered it
- Virtual JP: block=false

On NO or JP block: BLOCKED receipt. Email jurand@otsbroker.com. No code written.

---

## G1 — DESIGN_ALIGNMENT

Purpose: Does this scope align with project goals and architecture rules?

Virtual JP reviews scope against full project memory.

Evidence required for YES:
- Scope does not contradict any rule in CLAUDE.md
- Scope is not redundant with completed work (per recent receipts)
- Scope fits the 4-week inbox focus
- No architecture rule violation (PostgreSQL, no dark theme, OCR files locked, etc.)

On NO: Session terminates. BLOCKED receipt.

---

## G2 — RISK_ANALYSIS

Purpose: What else breaks if we make this change?

Virtual Michal reviews risk analysis for thoroughness.

Evidence required for YES:
- Machine grep: which other files import the files being changed
- Affected services identified
- Migration requirement stated (yes/no)
- Service restart list
- Rollback complexity rated
- Virtual Michal: block=false (risk analysis is thorough enough)

On NO: Implement risk mitigations first. Do not proceed to G3 until G2 is YES.

---

## G3 — IMPLEMENTATION

Purpose: Claude writes the code.

Scope: Only files listed in G0. No additions. No "while I'm here" changes.

If a file not in G0 scope needs to be touched: STOP. Declare the addition. State acceptance criteria. Wait for confirmation.

File locking: Before any file is written, gate_v2/file_lock.py acquires an atomic mkdir lock on every file in scope. If any file is already locked by another session, G3 returns LOCK_CONFLICT (BLOCKED). Locks auto-expire at 30 minutes. Use force_unlock() for debug only.

After implementation: run run_post_implementation_gates().

---

## G4 — CODE_QUALITY

Purpose: Is the implementation best-in-class, tested, and secure?

Virtual Michal reviews every changed file.

Michal runs tiered 30/30 gate:
- NEW_FEATURE (new endpoint / screen / schema change): 30/30 — 10 FE + 10 BE + 10 Risk checks
- BUG_FIX (<20 lines, no schema change): 10-check subset (tests, validation, observability, secrets, rollback)
- SMOKE (typo / config tweak): smoke test only

Michal must declare tier before reviewing. Output is a JSON gate table (CHECK_ID | PASS/FAIL | ARTEFACT_PATH | NOTES). Anything below required pass count = hard NO, redo required. Max 3 redo cycles then escalate to JP.

Michal model: deepseek-reasoner for full gates (600s), deepseek-chat for incremental (120s, 2000 chars/file).

Sycophancy block — Michal is forbidden from: "should work", "looks good", "appears to", "mostly compliant", "acceptable for now", "fix later".

Evidence required for YES:
- python3 -m py_compile passes for all changed .py files
- Virtual Michal: block=false
- GPT-4o-mini binary gate: YES

On NO: Fix Michal's specific objections. Re-run G4. Do not deploy until YES.

---

## G5 — DEPLOY_VERIFY

Purpose: Is the change visible on hub.otsbroker.com?

Evidence required for YES:
1. Services running: systemctl is-active hub-mail-api → active
2. No errors in logs: journalctl -u hub-mail-api -n 30 | grep -i "traceback\|error" → empty
3. Baseline diff: curl of target URL before and after deploy, diff contains acceptance selector from G0

On NO: Rollback. BLOCKED receipt. Email jurand@otsbroker.com.

---

## G6 — SESSION_SEAL

Purpose: Commit, seal, email, update memory.

Virtual JP reviews: was the right thing built? Does outcome match project goals?
Virtual Michal reviews: is the receipt complete, honest, and standardised?

Evidence required for YES:
- Git commit hash recorded
- Receipt written in standard v2 JSON format via write_receipt()
- Virtual JP: block=false
- GPT-4o-mini binary gate: YES
- Email sent to jurand@otsbroker.com
- Memory update: gpt-4o-mini extracts new decisions/facts from session, appends to MEMORY.md under dated heading (non-blocking — failure logs to stderr, does not prevent seal)
- File locks released for all files in scope

```python
from gate_v2.run_gate import seal_session
result = seal_session(
    session_id=SESSION_ID,
    session_summary="one paragraph summary",
    commit_hash=COMMIT_HASH,
    scope=G0_SCOPE,
    artifacts=ARTIFACT_PATHS
)
```

---

## BLOCKED RECEIPT FORMAT

Every blocked session writes:

```json
{
  "schema_version": "2.0",
  "session_id": "...",
  "gate": "G[N]",
  "decision": "BLOCKED",
  "decided_by": "...",
  "timestamp": "...",
  "scope": "...",
  "artifacts": [],
  "gate_results": {"G0": "?", "G1": "?", ...},
  "risk_report": {"affected_files": [], "affected_services": [], "migration_required": false},
  "test_results": {"lint": "SKIP", "unit_tests": "SKIP", "smoke_test": "SKIP"},
  "virtual_jp": {"block": false, "reason": ""},
  "virtual_michal": {"block": false, "quality_issues": []},
  "known_limits": [],
  "human_emailed": true,
  "commit_hash": ""
}
```

Path: /home/otsadmin/virtual/supervisors/receipts/{SESSION_ID}_BLOCKED.json

---

## RECEIPT STANDARDISATION

All receipts use schema v2.0. No exceptions.

Enforced by: /home/otsadmin/virtual/gate_v2/receipt_validator.py
The validator runs before every receipt write. A receipt that fails validation cannot be written.

No free-text receipts. No HTML receipts. No 10-character receipts.
One format. One schema. Always.

---

## REQUIRED OUTPUT — session start confirmation

After preflight and before G0:

```
SESSION READY v2
Session ID: [YYYYMMDD_HHMMSS]
Project: Hub
Pre-flight: git [ok/FAIL] | GPT-4o-mini [ok/FAIL] | DeepSeek [ok/FAIL] | Gemini [ok/FAIL]
Project memory: loaded [N chars MEMORY.md, N chars CLAUDE.md, N receipts]
Yesterday: [one sentence on what was last deployed and where visible]
Git status: [N uncommitted files / clean]
Gate sequence: FULL (new feature) or INCREMENTAL (continuation)
```

Then G0 begins immediately.

---

## ESCALATION TO 3-MODEL REVIEW

3-model review (GPT-5.5 + DeepSeek + Gemini) is used ONLY when:
- risk_class = "critical" (auth, security, data migration affecting all users)
- A gate returns NO twice consecutively
- Virtual JP and Virtual Michal disagree on a block decision

In all other cases: 1 primary + 1 challenger only.

---

## CHANGELOG

| Date | Change |
|---|---|
| 2026-04-30 | GPT-5.5 primary, GPT-5.4 fallback, hard stop if both unavailable |
| 2026-04-30 | Michal: 30/30 gate (10 FE + 10 BE + 10 Risk), JSON table output, tiered, sycophancy block |
| 2026-04-30 | Michal: deepseek-chat 120s for incremental, reasoner 600s for full |
| 2026-04-30 | G3 file locking: mkdir-atomic, 30min expire, GateLockConflict |
| 2026-04-30 | G6 memory update: gpt-4o-mini extracts decisions → MEMORY.md (non-blocking) |
| 2026-04-30 | Information isolation unit test: 5/5 pass, strip confirmed |
| 2026-04-30 | 7 legacy receipt files marked # gate-v2-exempt |
| 2026-04-29 | Protocol v2 built — DEPLOYED/BLOCKED binary outcomes, G0→G6 sequence |
