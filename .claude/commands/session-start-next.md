# hub-binary-gate-protocol-v1-NEXT.md
# System: next.otsbroker.com | Binary-Gate Self-Governing AI Development Protocol
# Version: 1.0 | Built: 2026-04-24 | Parallel to session-start-hub.md v2.0
# Stack: Next.js 14 / React 18 / TypeScript / SQLite / HMRC CTC API
# Authority: This file governs ALL AI development sessions on NEXT. It cannot be bypassed.

---

## RULE ZERO

Every session ends in exactly one state: **DEPLOYED** or **BLOCKED**.
No PARTIAL. No COMPLETE without a browser-verifiable change on next.otsbroker.com.

NEXT and Hub are **separate projects**. A session on NEXT cannot touch Hub files. A session on Hub cannot touch NEXT files. Cross-project changes are a hard stop.

---

## GATE SEQUENCE (strict linear — no skip, no parallel, no shortcut)

```
G0 SCOPE_LOCK → G1 PRE_STATE → G2 PLAN_APPROVE → G3 CODE_REVIEW → G4 DEPLOY_VERIFY → G5 DIFF_CONFIRM → G6 SESSION_SEAL
```

Every gate returns exactly: **YES** or **NO**
**NO at any gate = session stops immediately. Rollback executes. Human is emailed. BLOCKED receipt written.**

---

## ROLE LOCK (immutable for all NEXT sessions)

| Role | Model | Allowed | Forbidden |
|---|---|---|---|
| Designer | GPT-5.4 | Propose scope, answer challenges | Gate own proposals, write code |
| Challenger | DeepSeek Reasoner | Challenge proposals, gate G0/G2/G3 | Propose features, implement |
| Implementer | Claude / Sonnet | Write code, deploy, provide evidence | Approve, validate, or close gates |
| Gate Executor | GPT-5.4-mini | Run all binary gates, write receipts | Propose, implement |
| Reality Check | curl (mechanical) | Capture live state G1 + G5 | Implement, propose, gate |

---

## EXECUTION RULE — NO TIMEOUTS

All GPT-5.4 and DeepSeek Reasoner API calls: `timeout=600000` (10 minutes). Never default 120s.

---

## MANDATORY PRE-FLIGHT (blocks session if any fail)

```bash
# 1. Load secrets
source /home/otsadmin/next/.env.local 2>/dev/null
export $(grep -v '^#' /home/otsadmin/.secrets/hubv2.env | xargs) 2>/dev/null

# 2. Confirm project scope — NEXT only
echo "Project: NEXT (next.otsbroker.com)"
echo "Working directory: /home/otsadmin/next"
# HARD STOP if any Hub file paths appear in scope

# 3. Git status
cd /home/otsadmin/next
git status --short
git log --oneline -5
# STOP if uncommitted prior-session changes without explanation

# 4. Yesterday visibility check
# Q: What was built last session?
# Q: Where is it visible on next.otsbroker.com right now?
# Q: If not visible — why was it prioritised?

# 5. API health
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

# 6. NEXT service health
systemctl is-active next && echo "NEXT service: RUNNING" || echo "NEXT service: DEAD — fix before proceeding"
curl -sf https://next.otsbroker.com/login && echo "Site: REACHABLE" || echo "Site: UNREACHABLE"
```

---

## G0 — SCOPE_LOCK

**Purpose:** GPT-5.4 proposes exactly one user-visible change. DeepSeek challenges it. Scope is frozen.

**NEXT-specific constraints:**
- Target routes: `/declarations`, `/exports`, `/imports`, `/transit`, `/gvms`, `/ens`, `/login`
- No changes to HMRC submission logic without explicit scope approval (safety-critical)
- No changes to NextAuth or authentication paths without explicit scope approval
- SQLite database changes require migration script at `scripts/migrate.js`

**Evidence required for YES:**
- One-sentence scope: what changes in the browser, which route, which user role benefits
- File list: exact files to change (max 5; if more, split sessions)
- Acceptance selector: exact text or element visible after deploy
- Rollback plan: `git reset --hard HEAD~1 && npm run build && sudo systemctl restart next`
- DeepSeek has raised ≥1 concrete challenge and GPT-5.4 has answered it
- DeepSeek confirms scope is specific enough to falsify after deployment

**Gate question:** "Is there exactly one user-visible change defined on next.otsbroker.com, with exact files, exact acceptance evidence, exact rollback, and has DeepSeek confirmed it is falsifiable?"

**On NO:** Session terminates. BLOCKED receipt. Human emailed. No code written.

**GPT-5.4 call pattern:**
```python
import openai, os
client = openai.OpenAI(api_key=os.environ['OPENAI_API_KEY'])
r = client.chat.completions.create(
    model="gpt-5.4",
    messages=[
        {"role":"system","content":"You are the product designer for NEXT — a customs declaration portal for OTS Broker. Users are declarants and operators submitting HMRC customs declarations (exports, imports, transit, GVMS). Propose one user-visible improvement to the declaration workflow that can be built and deployed in this session. Be specific: which file changes, what element appears in the browser, which user benefits, what the rollback command is. Never touch HMRC submission logic unless explicitly scoped."},
        {"role":"user","content":"Session date: 2026-04-24. Last session: [FILL FROM PRE-FLIGHT]. Design the next single deployable improvement to next.otsbroker.com."}
    ],
    max_completion_tokens=1000
)
scope_proposal = r.choices[0].message.content
```

---

## G1 — PRE_STATE

**Purpose:** Capture current browser-visible state of the target page.

```bash
SESSION_ID=$(date +%Y%m%d_%H%M%S)_NEXT
mkdir -p /tmp/session_${SESSION_ID}
TARGET_URL="https://next.otsbroker.com/[route-from-G0]"

curl -sf -o /tmp/session_${SESSION_ID}/g1_baseline.html "${TARGET_URL}" && \
  sha256sum /tmp/session_${SESSION_ID}/g1_baseline.html > /tmp/session_${SESSION_ID}/g1_hash.txt && \
  echo "G1: YES — baseline captured at $(date)" || echo "G1: NO — site unreachable"
```

**Gate question:** "Has the current browser-visible state of the target NEXT page been captured with timestamp and content hash?"

**On NO:** Infrastructure problem. Do not write code. Email jurand@otsbroker.com.

---

## G2 — PLAN_APPROVE

**Purpose:** Claude writes file-by-file implementation plan. DeepSeek reviews.

**NEXT-specific additions to plan:**
- If touching database: name the migration file and migration content
- If touching HMRC API paths: explicitly state "HMRC submission logic untouched"
- If touching auth: requires explicit scope approval before proceeding

**Evidence required for YES:**
- File-by-file list: filename → what changes → why
- Every service requiring restart named (`next` systemd service)
- Database migration included if schema changes
- Rollback commands exact
- DeepSeek confirms plan produces G0 outcome, touches only scoped files

**Gate question:** "Does the file-by-file plan produce the G0 scoped outcome, touch only scoped files, include migration if needed, and have a credible rollback?"

---

## G3 — CODE_REVIEW

**Purpose:** Claude writes code. DeepSeek reviews actual git diff.

**NEXT-specific review checklist:**
- No hardcoded HMRC credentials or bearer tokens
- No changes to `src/lib/hmrc/` without explicit safety scope
- TypeScript types maintained (no `any` additions without justification)
- `next build` must succeed (TypeScript compiler, not just eslint)

**Evidence required for YES:**
- Complete `git diff`
- DeepSeek confirms: diff matches G2 plan, no out-of-scope changes, no credential exposure, no HMRC logic touched without scope

**Gate question:** "Does the actual code diff match the approved plan with no out-of-scope changes, no credential exposure, and no safety-critical HMRC logic changes?"

**On NO:** Claude revises once. DeepSeek re-reviews. Second NO: `git reset --hard`. BLOCKED receipt. No deployment.

---

## G4 — DEPLOY_VERIFY

**Purpose:** Claude deploys. GPT-5.4-mini runs mechanical checks.

**NEXT deploy sequence:**
```bash
cd /home/otsadmin/next

# Build (TypeScript compile + Next.js build)
npm run build
BUILD_EXIT=$?
echo "Build exit: ${BUILD_EXIT}"
[ $BUILD_EXIT -ne 0 ] && echo "G4: NO — build failed" && exit 1

# Run migrations if DB touched
# node scripts/migrate.js  (only if G2 plan includes migration)

# Restart service
sudo systemctl restart next
sleep 3
```

**G4 checks (ALL must pass):**
```bash
systemctl is-active next && echo "SERVICE: PASS" || echo "SERVICE: FAIL"
curl -sf https://next.otsbroker.com/login && echo "HEALTH: PASS" || echo "HEALTH: FAIL"
journalctl -u next -n 30 --no-pager | grep -i "error\|unhandled\|traceback" && echo "LOGS: FAIL" || echo "LOGS: PASS"
```

**Gate question:** "Did npm run build succeed, is the next service active, does the login page return 2xx, and are logs free of errors?"

**On NO:** Execute rollback: `git reset --hard HEAD~1 && npm run build && sudo systemctl restart next`. BLOCKED receipt. Human emailed.

---

## G5 — DIFF_CONFIRM (THE CRITICAL GATE)

**Purpose:** Verify next.otsbroker.com now shows the change.

```bash
curl -sf -o /tmp/session_${SESSION_ID}/g5_post.html "${TARGET_URL}"
diff /tmp/session_${SESSION_ID}/g1_baseline.html /tmp/session_${SESSION_ID}/g5_post.html \
  > /tmp/session_${SESSION_ID}/g5_diff.txt

ACCEPTANCE_SELECTOR="[FILL FROM G0]"
grep -q "${ACCEPTANCE_SELECTOR}" /tmp/session_${SESSION_ID}/g5_post.html && \
  echo "G5: YES — acceptance selector found in live page" || \
  echo "G5: NO — acceptance selector NOT found in live page"
```

**Gate question:** "Is the exact acceptance selector from G0 present in the live next.otsbroker.com page, verified against the G1 baseline?"

**On NO:** PHANTOM DEPLOY — services running but nothing changed. Rollback. BLOCKED receipt with HIGH PRIORITY flag. Human emailed immediately.

---

## G6 — SESSION_SEAL

**Purpose:** Commit, seal, close with full evidence.

```bash
cd /home/otsadmin/next
git add [scoped files from G2 only]
git commit -m "session: ${SESSION_ID} — [G0 scope in one line]"
COMMIT_HASH=$(git rev-parse HEAD)
echo "COMMIT: ${COMMIT_HASH}"
```

**Evidence required for YES:**
- Git commit SHA recorded
- G5 returned YES
- Session receipt written to `/home/otsadmin/virtual/supervisors/receipts/[SESSION_ID]_NEXT_[task].json`
- Human notification email sent: session ID, commit hash, target URL, diff path

**Gate question:** "Does a git commit hash exist, did G5 pass, and has a complete receipt been written and human notified?"

---

## NEXT-SPECIFIC HARD STOPS

Stop immediately and email jurand@otsbroker.com if:

1. Scope touches `src/lib/hmrc/` — HMRC submission logic (safety-critical, needs explicit approval)
2. Scope touches NextAuth configuration — authentication (security-critical)
3. Scope touches Hub files (`/home/otsadmin/deployed/hub_mail_rebuild/`) — cross-project spill
4. `npm run build` fails and cannot be fixed within session — do not deploy broken build
5. Database migration would drop or alter existing columns — requires backup first

---

## BLOCKED RECEIPT FORMAT

```json
{
  "session_id": "[SESSION_ID]",
  "project": "NEXT",
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

---

## DEPLOYED RECEIPT FORMAT

```json
{
  "session_id": "[SESSION_ID]",
  "project": "NEXT",
  "status": "DEPLOYED",
  "g0_scope": "[one-sentence scope]",
  "files_changed": ["file1", "file2"],
  "commit_hash": "[SHA]",
  "target_url": "https://next.otsbroker.com/[route]",
  "acceptance_selector": "[text/element verified]",
  "g1_baseline_path": "/tmp/session_[ID]/g1_baseline.html",
  "g5_post_path": "/tmp/session_[ID]/g5_post.html",
  "gates": {
    "G0": "YES", "G1": "YES", "G2": "YES", "G3": "YES",
    "G4": "YES", "G5": "YES", "G6": "YES"
  },
  "human_emailed": true,
  "timestamp": "[ISO timestamp]"
}
```

---

## PRODUCT VISION — NEXT

*Read before designing any NEXT feature.*

**Primary users:** Customs declarants and operators submitting declarations to HMRC.

**Core jobs NEXT must do:**
1. Submit accurate HMRC declarations (exports, imports, transit, GVMS) without errors
2. Track declaration status clearly — what is submitted, pending, accepted, rejected
3. Surface HMRC error codes in plain language so operators can fix and resubmit
4. Reduce manual re-entry and copy-paste between HMRC systems

**What "excellent" means for NEXT:**
- Declarant can submit a declaration in under 5 minutes without referring to external docs
- HMRC rejection reasons are shown in plain English with clear fix instructions
- Status of every active declaration is visible at a glance
- Common errors are caught before submission, not after

**What NEXT must not become:**
- Not a duplicate of Hub's inbox/mail features
- Not a generic document management system
- Not a system requiring HMRC knowledge to operate

---

## REQUIRED OUTPUT — session start confirmation

```
SESSION READY
Session ID: [YYYYMMDD_HHMMSS]_NEXT
Project: NEXT (next.otsbroker.com)
Pre-flight: git ✓ | GPT-5.4 ✓ | DeepSeek ✓ | next service ✓
Yesterday visibility: [one sentence on last deployed change and where visible]
Git status: [N uncommitted files / clean]
Cross-project check: Hub files untouched ✓
Quality bar: G5 browser-verified deployment required
```
