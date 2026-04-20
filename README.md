# alfred_ — Execution Decision Layer

A prototype of alfred_'s execution decision engine: given a proposed action and conversation context, the system decides whether to execute silently, execute and notify, confirm first, ask a clarifying question, or refuse.

**Live demo:** [add deployed URL here]  
**Stack:** Vanilla HTML/JS frontend + Node/Express backend + Anthropic API (claude-sonnet-4)

## Run locally

1. Install dependencies:
   - `npm install express cors`
2. Set your Anthropic key:
   - add it to `.env` as `ANTHROPIC_API_KEY=your_key_here`
3. Start backend:
   - `node server.js`
4. Open the UI:
   - open `alfred_decision_layer.html` in your browser
5. In the UI, click `Decide` to send requests to:
   - `http://localhost:3001/decide`

---

## Key design decisions

1. **Safety over automation rate**  
   I optimize for avoiding irreversible mistakes, even if that means asking for extra confirmation.
2. **Deterministic rules before model judgment**  
   Anything auditable and predictable (risk tier, missing entities, policy checks, hesitation/conflict patterns) is computed in code first.
3. **Conversation-first interpretation**  
   The latest message is never treated in isolation; prior hesitation in history can override a short "yes/send it."
4. **Single source of truth on backend**  
   The backend owns signal computation, prompt construction, model invocation, and fallback behavior. The UI only visualizes pipeline outputs.
5. **Failure handling as product behavior**  
   Timeout, malformed output, and API errors are explicit product states with safe fallback messaging, not hidden system errors.

### Decision contract

Given `{ action, latest_message, conversation_history, user_context }`, the system always returns one of:
- `execute_silent`
- `execute_notify`
- `confirm`
- `clarify`
- `refuse`

Guarantee: on uncertainty or pipeline failure, default to a safe decision (`confirm`), never silent execution.

### Policy table (deterministic overrides)

| Condition | Decision | Why |
|-----------|----------|-----|
| Bulk destructive action (e.g., delete all emails) | `refuse` | Irreversible, policy-prohibited |
| Missing critical entity (recipient/time/referent) | `clarify` | Avoid guessing when intent is underspecified |
| Hesitation in history + vague confirmation now | `confirm` | Prevent unsafe recency bias after prior stop signal |

---

## How it works

The system is a two-stage pipeline: deterministic signal computation runs first, and the LLM only runs if no deterministic rule fires.

### Stage 1 — Deterministic signal computation

Before the model is called, the following signals are computed in code:

| Signal | What it detects | Why it matters |
|--------|----------------|----------------|
| **Action category** | Maps action text to a category (email_send, calendar_delete, bulk_delete, etc.) | Sets the risk tier and reversibility flag without LLM involvement |
| **Risk tier** | low / medium / high / critical derived from category | Drives whether silent execution is even on the table |
| **Reversibility** | Whether the action can be undone | Irreversible + high risk → never execute silently |
| **Hesitation detection** | Scans full conversation history for patterns like "hold off", "wait", "actually", "never mind" | Catches cases where the user paused a flow before re-confirming |
| **Conflict detection** | Hesitation followed by a vague confirmation ("Yep", "Send it", "Do it") | The core trap: don't treat a bare "yes" as sufficient after a stop signal |
| **Entity resolution** | Checks for missing who/what/when (recipient, time, referent) | Clarify before acting, never guess |
| **Policy violation** | Bulk destructive actions, or high-risk actions from low-trust users | Hard block — never reaches the LLM |

Three deterministic forced decisions skip the LLM entirely:
- `refuse` — policy violation detected
- `clarify` — required entity is missing
- `confirm` — conflict detected (hesitation + vague confirmation)

### Stage 2 — LLM decision (claude-sonnet-4)

If no forced decision fires, the LLM receives:
- All computed signals as ground truth (it does not re-derive them)
- Full conversation history
- The proposed action and latest message
- Decision rules as explicit constraints

The model decides between the 5 options and returns structured JSON with a rationale, suggested message, confidence score, and which signals it weighted.

### What the model decides vs. what code computes

| Computed deterministically | Decided by LLM |
|---------------------------|---------------|
| Risk tier, reversibility | Nuanced intent disambiguation |
| Entity completeness | Tone calibration for suggested message |
| Hesitation/conflict detection | Confidence scoring |
| Policy violations | Edge cases not covered by rules |
| Forced overrides | Final decision when signals are ambiguous |

The key design principle: **code handles what is knowable, the LLM handles what requires judgment.** Signal computation is deterministic and auditable. The LLM is told to treat those signals as ground truth, not re-derive them.

---

## Prompt design

The prompt structure:
1. **Role + task** — brief, no preamble
2. **Decision option definitions** — exact labels with tiebreaking guidance
3. **Deterministic signals** — injected as a pre-computed block, framed as "ground truth"
4. **Conversation history** — full, formatted as `[USER]` / `[ASSISTANT]` turns
5. **Latest message + proposed action** — separated so the model doesn't over-index on recency
6. **Decision rules** — explicit, especially the hesitation+confirmation trap
7. **Output format** — strict JSON schema, no markdown fences

The prompt explicitly instructs the model: *never treat the latest message in isolation*. The full history is the primary signal.

---

## Failure modes

### Handled in this prototype

| Failure | Behavior |
|---------|----------|
| LLM timeout (>10s) | Fall back to `confirm` — never silent-execute on uncertainty |
| Malformed model output | Fall back to `confirm` |
| Missing critical context | Deterministic entity check fires `clarify` before LLM is called |
| Policy violation | Hard `refuse` from deterministic layer — LLM never consulted |
| Unexpected API error | Fall back to `confirm` with error surfaced in UI |

Default safe behavior: **any pipeline failure defaults to `confirm`, never `execute_silent`**. The asymmetry is intentional — a missed execution is annoying; an incorrect irreversible action is damaging.

### Known gaps / expected failure modes in production

- **Entity resolution is shallow** — currently regex-based; would miss complex cases like "send it to the person we met Tuesday"
- **Trust is static** — trust level is user-set; in production it should be dynamic (recent behavior, action history, anomaly detection)
- **No rate limiting on confirmations** — a user who keeps saying "yes, do it" after every hesitation could still be asked to confirm repeatedly; needs a confirmation fatigue model
- **Category detection is heuristic** — free-form action strings are messy; a classifier would be more robust
- **No memory across sessions** — context is per-conversation only

---

## Scenario coverage

| Scenario | Expected decision | Why |
|----------|------------------|-----|
| "Remind me to call mom at 5pm" | execute_silent | Low risk, reversible, clear intent |
| "What's on my calendar today?" | execute_silent | Read-only, no side effects |
| "Yep, send it" after legal hold | confirm | Conflict detection: hesitation + vague confirmation |
| "Cancel the CEO meeting" while stressed | confirm | Irreversible + time-sensitive + no prior explicit confirmation |
| "Delete all my emails from last year" | refuse | Policy violation — bulk destructive action |
| "Send it" with no prior context | clarify | Entity resolution: missing recipient/referent |

---

## How I'd evolve this as alfred_ gains riskier tools

### Short term (next 6 months)
- **Action schema**: move from free-text action strings to typed action objects (`{type: "email.send", recipient: "...", draft_id: "..."}`) — makes signal computation reliable and adds a clear place to attach metadata
- **Dynamic trust scoring**: model trust as a function of recency, action history, reversal frequency, and domain (financial vs. scheduling)
- **Tiered confirmation UX**: not all confirms look alike — a low-stakes confirm is a quick "going ahead, is that ok?" while a high-stakes confirm is an explicit approval flow with preview
- **Undo support where possible**: prefer reversible actions where equivalent (archive vs. delete; draft vs. send) and surface undo windows
- **Better entity resolution**: structured entity extraction with a dedicated pass, not regex

### Medium term
- **Risk forecasting**: model downstream consequences of an action, not just immediate risk (sending a discount email is medium risk in isolation but high risk if the deal is in final negotiation)
- **Multi-step planning review**: for compound tasks ("clear my afternoon and reschedule everything to next week"), show the full plan before executing any step
- **Audit log**: every decision logged with full pipeline trace — essential for debugging and trust
- **Feedback loop**: when the user corrects alfred_ ("no, I said don't send it"), that signal feeds back into trust and future hesitation detection

### Longer term
- **Fine-tuned decision model**: once there's enough logged decisions + user corrections, distill the decision logic into a smaller, faster, more calibrated model
- **Tool-specific risk models**: different tools have different risk surfaces; calendar tools have lower stakes than financial tools — each needs its own policy layer
- **Delegation levels**: users should be able to set explicit permissions per tool, per contact, per time window

---

## What I chose not to build

- Authentication / user management
- Persistent conversation storage
- Fine-grained entity extraction (NLP-grade)
- A confirmation UX that's differentiated by risk level (all confirms look the same)
- Analytics / decision logging

These were all deliberate scope cuts to stay within the timebox. The prototype demonstrates the decision logic and pipeline transparency — which is the core of the problem.
