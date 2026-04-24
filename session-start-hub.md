# hub-binary-gate-protocol-v1.md
# System: hub.otsbroker.com | Binary-Gate Self-Governing AI Development Protocol
# Version: 2.0 | Built: 2026-04-24 | Supersedes: session-start-hub.md (FAILED — scored 1-18/100 by GPT-5.4, DeepSeek, Opus)
# Designed by: GPT-5.4 + DeepSeek Reasoner + Opus 4 in parallel cross-analysis
# Authority: This file governs ALL AI development sessions on Hub. It cannot be bypassed.

---

## RULE ZERO — the reason this file exists

The previous protocol failed for 6 weeks because AI produced files while hub.otsbroker.com remained unchanged.
AI output ≠ deployed reality. This protocol makes that gap impossible to ignore.

**Every session ends in exactly one state: DEPLOYED or BLOCKED.**
There is no PARTIAL. There is no COMPLETE without a browser-verifiable change on hub.otsbroker.com.

---

## GATE SEQUENCE (strict linear — no skip, no parallel, no shortcut)

```
G0 SCOPE_LOCK → G1 PRE_STATE → G2 PLAN_APPROVE → G3 CODE_REVIEW → G4 DEPLOY_VERIFY → G5 DIFF_CONFIRM → G6 SESSION_SEAL
```

Every gate returns exactly: **YES** or **NO**
**NO at any gate = session stops immediately. Rollback executes. Human is emailed. BLOCKED receipt is written.**

---

## ROLE LOCK (immutable for all sessions)

| Role | Model | Allowed | Forbidden |
|---|---|---|---|
| Designer | GPT-5.4 | Propose scope, answer challenges | Gate own proposals, write code |
| Challenger | DeepSeek Reasoner | Challenge proposals, gate G0/G2/G3 | Propose features, implement |
| Architecture | Opus 4 | Architecture review (if invoked) | Gate implementation |
| Implementer | Claude / Sonnet | Write code, deploy, provide evidence | Approve, validate, or close gates |
| Gate Executor | GPT-5.4-mini | Run all binary gates, write receipts | Propose, implement |
| Reality Check | Gemini / curl | Capture live browser state G1 + G5 | Implement, propose, gate |

**Hard rule:** The model that produces an artifact cannot be the model that gates that artifact.
If producer == gater: the protocol is broken. Fix it before proceeding.

---

## EXECUTION RULE — NO TIMEOUTS

All GPT-5.4, DeepSeek Reasoner, and Opus API calls: use Bash tool with `timeout=600000` (10 minutes).
Never default 120s — reasoning models need time.

---

## AUTO-UPDATE (runs silently at session start)

```bash
[ -n "$GITHUB_TOKEN" ] && curl -sL -H "Authorization: token ${GITHUB_TOKEN}" \
  https://raw.githubusercontent.com/jurand1991/session-start-hub/main/.claude/commands/session-start-hub.md \
  -o /tmp/session-start-hub-update.md 2>/dev/null \
  && grep -q "SESSION READY" /tmp/session-start-hub-update.md \
  && cp /tmp/session-start-hub-update.md ~/.claude/commands/session-start-hub.md &
```

---

## MANDATORY PRE-FLIGHT (runs before G0 — blocks session if any fail)

```bash
# 1. Load secrets
export $(grep -v '^#' ~/.secrets/hubv2.env | xargs)

# 2. Git status — HARD STOP if project untracked
cd /home/otsadmin
git status --short
git log --oneline -5
# STOP if uncommitted changes from prior session exist without explanation

# 3. Yesterday visibility check — answer in 1 sentence each
# Q: What was built last session?
# Q: Where is it visible in the UI right now?
# Q: If not visible: why was it prioritised over visible work?

# 4. API health checks
python3 -c "
import openai, os, requests
client = openai.OpenAI(api_key=os.environ['OPENAI_API_KEY'])
r = client.chat.completions.create(model='gpt-5.4-mini', messages=[{'role':'user','content':'ping'}], max_completion_tokens=5)
print('GPT-5.4-mini: OK')
r2 = requests.post('https://api.deepseek.com/v1/chat/completions',
  headers={'Authorization': f'Bearer {os.environ[\"DEEPSEEK_API_KEY\"]}'},
  json={'model':'deepseek-chat','messages':[{'role':'user','content':'ping'}],'max_tokens':5}, timeout=30)
print('DeepSeek: OK' if r2.ok else f'DeepSeek: FAIL {r2.status_code}')
"
```

If pre-flight fails: **STOP. Do not proceed to G0. Email jurand@otsbroker.com explaining what failed.**

---

## G0 — SCOPE_LOCK

**Purpose:** GPT-5.4 proposes exactly one user-visible change. DeepSeek challenges it. Scope is frozen.

**Gated by:** DeepSeek Reasoner (GPT-5.4-mini calls the final binary gate)

**Evidence required for YES:**
- One-sentence scope: what changes in the browser, which page, which user role benefits
- File list: exact files to change (max 5; if more, split sessions)
- Acceptance selector: exact text or element that will be present/changed after deploy
- Rollback plan: exact commands to undo
- DeepSeek has raised ≥1 concrete challenge and GPT-5.4 has answered it
- DeepSeek confirms the scope is specific enough to falsify after deployment

**Gate question:** "Is there exactly one user-visible change defined, with exact files, exact acceptance evidence, exact rollback, and has DeepSeek confirmed it is falsifiable?"

**On NO:** Session terminates. BLOCKED receipt written. Human emailed: "BLOCKED at G0: Scope not locked — [DeepSeek's specific objection]." No code written.

**GPT-5.4 call pattern:**
```python
import openai, os, json
client = openai.OpenAI(api_key=os.environ['OPENAI_API_KEY'])
r = client.chat.completions.create(
    model="gpt-5.4",
    messages=[{"role":"system","content":"You are the product designer for Hub. Propose one user-visible inbox improvement that can be built and deployed in this session. Be specific: which file changes, what element appears in the browser, which user role benefits, what the rollback command is."},
              {"role":"user","content":"Session date: 2026-04-24. Last session: [FILL FROM PRE-FLIGHT]. Design the next single deployable improvement."}],
    max_completion_tokens=1000
)
scope_proposal = r.choices[0].message.content
```

**DeepSeek challenge call pattern:**
```python
import requests, os
resp = requests.post(
    "https://api.deepseek.com/v1/chat/completions",
    headers={"Authorization": f"Bearer {os.environ['DEEPSEEK_API_KEY']}"},
    json={"model":"deepseek-reasoner",
          "messages":[{"role":"system","content":"You challenge AI development proposals. Find the weakest assumption. Confirm or reject whether the scope is specific enough to verify after deployment."},
                      {"role":"user","content":f"Challenge this proposal:\n{scope_proposal}"}],
          "max_tokens":800},
    timeout=600
)
challenge = resp.json()["choices"][0]["message"]["content"]
```

**GPT-5.4-mini binary gate:**
```python
gate_question = f"""Proposal:\n{scope_proposal}\n\nDeepSeek Challenge Response:\n{challenge}\n\n
Does the proposal define exactly one user-visible change, with exact files, exact browser acceptance evidence, exact rollback, and has DeepSeek confirmed it is specific enough to falsify? Answer YES or NO only."""

r = client.chat.completions.create(
    model="gpt-5.4-mini",
    messages=[{"role":"user","content":gate_question}],
    response_format={"type":"json_schema","json_schema":{"name":"gate","strict":True,"schema":{"type":"object","properties":{"decision":{"type":"string","enum":["YES","NO"]}},"required":["decision"],"additionalProperties":False}}},
    max_completion_tokens=10
)
result = json.loads(r.choices[0].message.content)
# If NO → stop immediately
```

---

## G1 — PRE_STATE

**Purpose:** Capture current browser-visible state of the target page as baseline evidence.

**Gated by:** Mechanical (curl + content hash) — no AI judgment, just capture

**Evidence required for YES:**
- HTTP GET to the exact target URL (hub.otsbroker.com/[route]) returns 2xx
- Response body saved to `/tmp/session_[ID]/g1_baseline.html`
- SHA256 hash of body recorded
- Timestamp recorded
- If site unreachable: G1 = NO (infrastructure problem, not code problem)

```bash
SESSION_ID=$(date +%Y%m%d_%H%M%S)
mkdir -p /tmp/session_${SESSION_ID}
TARGET_URL="https://hub.otsbroker.com/[route-from-G0]"
curl -sf -o /tmp/session_${SESSION_ID}/g1_baseline.html "${TARGET_URL}" && \
  sha256sum /tmp/session_${SESSION_ID}/g1_baseline.html > /tmp/session_${SESSION_ID}/g1_hash.txt && \
  echo "G1: YES — baseline captured at $(date)" || echo "G1: NO — site unreachable"
```

**Gate question:** "Has the current browser-visible state of the target page been captured with timestamp, content hash, and the page responds 2xx?"

**On NO:** Session terminates. BLOCKED receipt. Human emailed. This is an infrastructure problem — do not code until resolved.

---

## G2 — PLAN_APPROVE

**Purpose:** Claude writes a file-by-file implementation plan. DeepSeek reviews it.

**Gated by:** DeepSeek Reasoner (GPT-5.4-mini calls binary gate)

**Evidence required for YES:**
- File-by-file list: filename → what changes → why
- Every service requiring restart is named
- Rollback commands are exact (not "revert the changes" — actual commands)
- DeepSeek confirms: plan produces G0 outcome, touches only scoped files, rollback is credible

**Gate question:** "Does the file-by-file plan, as reviewed by DeepSeek, fully and exclusively produce the G0 scoped outcome with a credible rollback?"

**On NO:** Claude revises plan once. DeepSeek reviews again. If second NO: session terminates. BLOCKED receipt. Human emailed.

---

## G3 — CODE_REVIEW

**Purpose:** Claude writes the code. DeepSeek reviews the actual git diff.

**Gated by:** DeepSeek Reasoner (GPT-5.4-mini calls binary gate)

**Evidence required for YES:**
- Complete `git diff` (actual diff, not summary)
- DeepSeek confirms: diff matches G2 plan, no out-of-scope changes, no hardcoded secrets, no destructive DB operations without transactions
- DeepSeek confirms: implementation is complete, not partial

**Gate question:** "Does the actual code diff match the approved plan with no out-of-scope changes and no security violations?"

**On NO:** Claude revises once. DeepSeek reviews new diff. If second NO: session terminates. `git reset --hard`. BLOCKED receipt. Human emailed. **No deployment occurs.**

---

## G4 — DEPLOY_VERIFY

**Purpose:** Claude deploys. GPT-5.4-mini runs mechanical checks.

**Gated by:** GPT-5.4-mini (mechanical command output only)

**Evidence required for YES (ALL must pass):**
1. `systemctl status hub-mail-api` → `active (running)`
2. `curl -sf https://hub.otsbroker.com/health` → 2xx response (or relevant health endpoint)
3. `journalctl -u hub-mail-api -n 30 --no-pager` → no Python traceback or unhandled exception
4. If database touched: `psql -U postgres hub -c "SELECT 1"` → success

```bash
# Deploy commands (from G2 plan):
cd /home/otsadmin/deployed/hub_mail_rebuild/frontend && npm run build
sudo systemctl restart hub-mail-api

# G4 checks:
systemctl is-active hub-mail-api && echo "SERVICE: PASS" || echo "SERVICE: FAIL"
curl -sf https://hub.otsbroker.com/health && echo "HEALTH: PASS" || echo "HEALTH: FAIL"
journalctl -u hub-mail-api -n 30 --no-pager | grep -i "traceback\|error\|exception" && echo "LOGS: FAIL" || echo "LOGS: PASS"
```

**Gate question:** "Are all affected services running, all affected endpoints returning 2xx, and logs free of exceptions?"

**On NO:** Execute rollback from G2. Re-run G4 checks against rolled-back state. BLOCKED receipt. Human emailed: "BLOCKED at G4: Deploy failed — [which check failed]. Rollback executed."

---

## G5 — DIFF_CONFIRM (THE CRITICAL GATE)

**Purpose:** Verify hub.otsbroker.com now shows the change. This is the gate whose absence caused the 6-week failure.

**Gated by:** Mechanical comparison + GPT-5.4-mini judgment on content diff

**Evidence required for YES:**
- New HTTP GET to same URL as G1 → response saved to `/tmp/session_[ID]/g5_post.html`
- Content diff between G1 baseline and G5 post-deploy
- The diff must contain the exact acceptance selector/text defined in G0
- If no diff exists: NO (phantom deploy — code ran, nothing changed)
- If diff exists but doesn't match G0 acceptance evidence: NO (wrong thing changed)

```bash
curl -sf -o /tmp/session_${SESSION_ID}/g5_post.html "${TARGET_URL}"
diff /tmp/session_${SESSION_ID}/g1_baseline.html /tmp/session_${SESSION_ID}/g5_post.html > /tmp/session_${SESSION_ID}/g5_diff.txt

# Gate: does the diff contain the acceptance selector from G0?
ACCEPTANCE_SELECTOR="[FILL FROM G0]"
grep -q "${ACCEPTANCE_SELECTOR}" /tmp/session_${SESSION_ID}/g5_post.html && echo "G5: YES" || echo "G5: NO — acceptance selector not found in live page"
```

**Gate question:** "Is the exact acceptance selector/text from G0 present in the live hub.otsbroker.com page, verified by content comparison with the G1 baseline?"

**On NO:** This is a critical failure. Code deployed, services run, nothing visible changed. Execute rollback. BLOCKED receipt with label "PHANTOM DEPLOY". Human emailed with HIGH PRIORITY flag. This failure pattern must be investigated before next session.

---

## G6 — SESSION_SEAL

**Purpose:** Commit, seal, and close the session with full evidence.

**Gated by:** GPT-5.4-mini

**Evidence required for YES:**
- Git commit exists with SHA recorded (`git rev-parse HEAD`)
- Commit message includes session ID
- G5 returned YES (checked in gate input)
- Session receipt written to `/home/otsadmin/virtual/supervisors/receipts/[SESSION_ID]_[task].json`
- Human notification email sent with: session ID, commit hash, target URL, G5 screenshot/diff path

```bash
cd /home/otsadmin
git add [scoped files from G2 only]
git commit -m "session: ${SESSION_ID} — [G0 scope in one line]"
COMMIT_HASH=$(git rev-parse HEAD)
echo "COMMIT: ${COMMIT_HASH}"
```

**Gate question:** "Does a git commit hash exist for this session, did G5 pass, and has a complete session receipt been written and human notified?"

**On NO:** Session cannot close as COMPLETE. BLOCKED receipt: "Session seal failed — [missing commit / missing G5 pass / incomplete receipt]." Human emailed. Session marked INCOMPLETE.

---

## BLOCKED RECEIPT FORMAT

Every blocked session must write this immediately:

```json
{
  "session_id": "[SESSION_ID]",
  "status": "BLOCKED",
  "blocked_at_gate": "G[N]",
  "gate_question": "[exact question that returned NO]",
  "reason": "[specific reason for NO]",
  "changes_reverted": true,
  "rollback_command": "[exact command run]",
  "human_emailed": true,
  "email_sent_to": "jurand@otsbroker.com",
  "timestamp": "[ISO timestamp]"
}
```

Path: `/home/otsadmin/virtual/supervisors/receipts/[SESSION_ID]_BLOCKED.json`

Email subject: `HUB PROTOCOL BLOCKED at G[N]: [one-line reason]`

---

## DEPLOYED RECEIPT FORMAT

Every successfully deployed session must write this:

```json
{
  "session_id": "[SESSION_ID]",
  "status": "DEPLOYED",
  "g0_scope": "[one-sentence scope]",
  "files_changed": ["file1", "file2"],
  "commit_hash": "[SHA]",
  "target_url": "https://hub.otsbroker.com/[route]",
  "acceptance_selector": "[text/element verified]",
  "g1_baseline_path": "/tmp/session_[ID]/g1_baseline.html",
  "g5_post_path": "/tmp/session_[ID]/g5_post.html",
  "g5_diff_path": "/tmp/session_[ID]/g5_diff.txt",
  "gates": {
    "G0": "YES", "G1": "YES", "G2": "YES", "G3": "YES",
    "G4": "YES", "G5": "YES", "G6": "YES"
  },
  "human_emailed": true,
  "timestamp": "[ISO timestamp]"
}
```

---

## PROTOCOL INTEGRITY CHECK (runs before every session)

GPT-5.4-mini runs this meta-check. All must return YES:

| Check | Question | Required |
|---|---|---|
| IC-1 | Does every gate have a gating model different from the producing model? | YES |
| IC-2 | Is G5 (DIFF_CONFIRM) present and un-bypassed? | YES |
| IC-3 | Is there a mechanism to email jurand@otsbroker.com on any NO? | YES |
| IC-4 | Is scope limited to exactly one user-visible change? | YES |
| IC-5 | Is the target URL hub.otsbroker.com (not localhost)? | YES |

If any check fails: **do not start the session.**

---

## DESIGN RULES (inherited from product vision)

- Every feature must be visible to an operator in under 10 seconds
- Default target: main inbox workflow (for next 4 weeks)
- Banned: hidden thread-panel-only features with no inbox visibility
- Banned: sessions that produce code without a G5-verified deployed change
- Banned: new `--outlook-*` CSS variables (use semantic tokens from design-tokens.css)
- Banned: new SQLite tables/columns (use PostgreSQL)
- OCR files protected: `/home/otsadmin/shared/document_reader.py` and `ocr_api.py` — do not touch

---

## REQUIRED OUTPUT — session start confirmation

After running pre-flight and before G0, output:

```
SESSION READY
Session ID: [YYYYMMDD_HHMMSS]
Project: Hub
Pre-flight: git ✓ | GPT-5.4 ✓ | GPT-5.4-mini ✓ | DeepSeek ✓
Yesterday visibility: [one sentence on what was last deployed and where it's visible]
Git status: [N uncommitted files / clean]
Protected zones: checked
Quality bar: G5 browser-verified deployment required
```

Then G0 begins immediately. GPT-5.4 proposes. DeepSeek challenges. Gate runs.
