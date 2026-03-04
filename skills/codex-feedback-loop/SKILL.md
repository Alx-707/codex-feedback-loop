---
name: codex-feedback-loop
description: "Autonomous Codex review loop — auto-detects code changes or plan context, sends to Codex, Claude analyzes & fixes, re-submits until approved (max 5 rounds). Use when the user says 'review', 'check my code/plan', or invokes /codex-feedback-loop."
argument-hint: "[plan] [model-override] [-- file-filter...]"
disable-model-invocation: true
context: fork
agent: general-purpose
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Codex Feedback Loop

Autonomous review loop: Codex reviews, Claude analyzes the feedback, fixes valid issues (code) or revises (plan), and re-submits until Codex approves. Max 5 rounds.

The user sees a summary of each round but does NOT need to intervene — Claude drives the entire loop.

## Arguments

- `plan` — Force plan review mode (optional). Without this, mode is auto-detected.
- `$0` — Model override (optional). Default: `gpt-5.2` (thinking: high). Example: `/codex-feedback-loop o4-mini`
- Anything after `--` — File path filters for code review. Example: `/codex-feedback-loop -- src/lib/ src/app/api/`

## Agent Instructions

### Step 0: Detect Mode

Determine whether to review **code** or a **plan**:

1. If the user passed `plan` as an argument → **Plan Mode**
2. Otherwise, check for code changes:
   ```bash
   BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main")
   CURRENT_BRANCH=$(git branch --show-current)
   if [ "$CURRENT_BRANCH" = "$BASE_BRANCH" ]; then
     HAS_DIFF=$(git diff HEAD --stat)
   else
     HAS_DIFF=$(git diff ${BASE_BRANCH}...HEAD --stat)$(git diff --stat)
   fi
   ```
3. If `HAS_DIFF` is non-empty → **Code Mode**
4. If `HAS_DIFF` is empty, check conversation context for a plan (from plan mode, or a plan discussed in chat) → **Plan Mode**
5. If neither → inform the user there's nothing to review and stop

Store the result as `REVIEW_MODE` (`code` or `plan`).

Tell the user which mode was detected:
```
**Mode:** ${REVIEW_MODE} review (auto-detected)
```
Or if forced via argument:
```
**Mode:** plan review (explicit)
```

### Step 1: Generate Session ID

```bash
REVIEW_ID=$(uuidgen | tr '[:upper:]' '[:lower:]' | head -c 8)
```

Temp file paths:
- **Code mode**: `/tmp/claude-diff-${REVIEW_ID}.patch`, `/tmp/claude-changed-${REVIEW_ID}.txt`, `/tmp/codex-review-${REVIEW_ID}.md`
- **Plan mode**: `/tmp/claude-plan-${REVIEW_ID}.md`, `/tmp/codex-review-${REVIEW_ID}.md`

### Step 2: Determine Model

If the user passed a model override (e.g., `/codex-feedback-loop o4-mini`), use that model. Otherwise default to `gpt-5.2`. Store as `CODEX_MODEL`.

Always use `--reasoning-effort high` regardless of model override, unless the user explicitly specifies otherwise.

Do NOT treat `plan` or file filter arguments as model override.

---

## Code Mode

### Step 3C: Capture Changes

```bash
if [ "$CURRENT_BRANCH" = "$BASE_BRANCH" ]; then
  DIFF_CMD="git diff HEAD"
else
  DIFF_CMD="git diff ${BASE_BRANCH}...HEAD"
fi
```

If the user provided file filters after `--`, append them to the diff command.

1. Write the diff to `/tmp/claude-diff-${REVIEW_ID}.patch`
2. Write the changed files list (with stats) to `/tmp/claude-changed-${REVIEW_ID}.txt`

**Include uncommitted changes** on a feature branch:
```bash
git diff ${BASE_BRANCH}...HEAD > /tmp/claude-diff-${REVIEW_ID}.patch
git diff >> /tmp/claude-diff-${REVIEW_ID}.patch
```

If the diff is very large (>2000 lines), warn the user and suggest file filters.

### Step 4C: Initial Review (Round 1)

```bash
codex exec \
  -m ${CODEX_MODEL} \
  --reasoning-effort high \
  -s read-only \
  -o /tmp/codex-review-${REVIEW_ID}.md \
  "$(cat ${CLAUDE_PLUGIN_ROOT}/skills/codex-feedback-loop/review-prompt.md)

The diff to review is at: /tmp/claude-diff-${REVIEW_ID}.patch
The changed files list is at: /tmp/claude-changed-${REVIEW_ID}.txt"
```

**Capture the Codex session ID** from the output line `session id: <uuid>`. Store as `CODEX_SESSION_ID`. You MUST use this exact ID to resume (do NOT use `--last`).

### Step 5C: Analyze Findings

1. Read `/tmp/codex-review-${REVIEW_ID}.md`
2. **Claude independently triages each finding**:
   - **Accept** — valid issue, will fix
   - **Skip** — false positive, style preference, or contradicts project conventions (document reason)

Present a round summary:
```
## Codex Review — Round N (model: ${CODEX_MODEL}, mode: code)

### Findings: X total (Y accepted, Z skipped)

**Accepted:**
- [file:line] severity: issue summary → will fix

**Skipped (with reasons):**
- [file:line] issue summary → reason for skipping

**Verdict:** REVISE / APPROVED
```

3. Check verdict:
   - **APPROVED** → Step 8 (Done)
   - **REVISE** with accepted findings → Step 6C (Fix)
   - **REVISE** but all findings skipped → treat as approved, note disagreements
   - No clear verdict but no actionable items → treat as approved
   - Max rounds (5) → Step 8 with remaining concerns

### Step 6C: Fix Code

For each accepted finding:
1. **Read the file** to understand full context
2. **Fix the issue** using the Edit tool — minimal, targeted changes
3. **Do NOT refactor surrounding code** or add unrelated improvements

```
### Fixes Applied (Round N)
- [file:line] what was fixed and why
```

**Critical rules:**
- Only fix issues that Codex flagged AND Claude agreed with
- If a fix requires architectural changes beyond PR scope, skip and note
- If a fix contradicts CLAUDE.md or .claude/rules/, skip and note
- Prefer the simplest correct fix over an elegant refactor

### Step 7C: Re-submit (Rounds 2-5)

Re-capture the diff:
```bash
if [ "$CURRENT_BRANCH" = "$BASE_BRANCH" ]; then
  git diff HEAD > /tmp/claude-diff-${REVIEW_ID}.patch
else
  git diff ${BASE_BRANCH}...HEAD > /tmp/claude-diff-${REVIEW_ID}.patch
  git diff >> /tmp/claude-diff-${REVIEW_ID}.patch
fi
```

Resume the Codex session:
```bash
codex exec resume ${CODEX_SESSION_ID} \
  "I've fixed the issues you identified. Here's what I changed:
[List specific fixes made]

[If any findings were skipped, explain why]

The updated diff is in /tmp/claude-diff-${REVIEW_ID}.patch.

Please re-review. Focus on:
1. Whether the fixes correctly address the original issues
2. Whether the fixes introduced any new problems
3. Any remaining issues in the original diff

If everything looks good, end with: VERDICT: APPROVED
If more changes are needed, end with: VERDICT: REVISE" 2>&1 | tee /tmp/codex-review-${REVIEW_ID}.md | tail -100
```

Go back to **Step 5C**.

If `resume` fails (session expired): fall back to fresh `codex exec` with prior round context.

---

## Plan Mode

### Step 3P: Capture Plan

Write the current plan from conversation context to `/tmp/claude-plan-${REVIEW_ID}.md`.

If there is no plan in context, ask the user what they want reviewed.

### Step 4P: Initial Review (Round 1)

```bash
codex exec \
  -m ${CODEX_MODEL} \
  --reasoning-effort high \
  -s read-only \
  -o /tmp/codex-review-${REVIEW_ID}.md \
  "Review the implementation plan in /tmp/claude-plan-${REVIEW_ID}.md. Focus on:
1. Correctness - Will this plan achieve the stated goals?
2. Risks - What could go wrong? Edge cases? Data loss?
3. Missing steps - Is anything forgotten?
4. Alternatives - Is there a simpler or better approach?
5. Security - Any security concerns?

Be specific and actionable. If the plan is solid, end with exactly: VERDICT: APPROVED
If changes are needed, end with exactly: VERDICT: REVISE"
```

**Capture `CODEX_SESSION_ID`** from output.

### Step 5P: Read Review & Check Verdict

1. Read `/tmp/codex-review-${REVIEW_ID}.md`
2. Present review:
```
## Codex Review — Round N (model: ${CODEX_MODEL}, mode: plan)

[Codex's feedback]
```
3. Check verdict:
   - **APPROVED** → Step 8 (Done)
   - **REVISE** → Step 6P (Revise)
   - No clear verdict but all positive → treat as approved
   - Max rounds (5) → Step 8

### Step 6P: Revise Plan

1. **Revise the plan** — address each issue Codex raised. Rewrite `/tmp/claude-plan-${REVIEW_ID}.md`.
2. Summarize:
```
### Revisions (Round N)
- [What was changed and why]
```

### Step 7P: Re-submit (Rounds 2-5)

```bash
codex exec resume ${CODEX_SESSION_ID} \
  "I've revised the plan based on your feedback. The updated plan is in /tmp/claude-plan-${REVIEW_ID}.md.

Here's what I changed:
[List specific changes]

Please re-review. If the plan is now solid, end with: VERDICT: APPROVED
If more changes are needed, end with: VERDICT: REVISE" 2>&1 | tee /tmp/codex-review-${REVIEW_ID}.md | tail -100
```

Go back to **Step 5P**.

If `resume` fails: fall back to fresh `codex exec` with prior round context.

---

## Shared: Final Result & Cleanup

### Step 8: Present Final Result

**Code mode — approved:**
```
## Codex Feedback Loop — Final (model: ${CODEX_MODEL}, mode: code)

**Status:** Approved after N round(s)
**Total findings:** X across all rounds
**Fixed:** Y | **Skipped:** Z

### Changes Made
- [file:line] description of fix

### Skipped Findings (if any)
- [file:line] issue → reason

---
**Code review complete. All fixes applied to working tree (unstaged).**
```

**Plan mode — approved:**
```
## Codex Feedback Loop — Final (model: ${CODEX_MODEL}, mode: plan)

**Status:** Approved after N round(s)

[Final approval message]

---
**Plan reviewed and approved by Codex. Ready for your approval to implement.**
```

**Max rounds reached (either mode):**
```
## Codex Feedback Loop — Final (model: ${CODEX_MODEL}, mode: ${REVIEW_MODE})

**Status:** Max rounds (5) reached — not fully approved

### Remaining Concerns
[List unresolved issues]

---
**Some concerns remain. Review the items above and decide whether to proceed.**
```

### Step 9: Cleanup

```bash
rm -f /tmp/claude-diff-${REVIEW_ID}.patch /tmp/claude-changed-${REVIEW_ID}.txt /tmp/claude-plan-${REVIEW_ID}.md /tmp/codex-review-${REVIEW_ID}.md
```

## Rules

- **Claude drives the entire loop autonomously** — the user observes but does not need to intervene
- **Analyze before acting** — never blindly apply all Codex suggestions. Validate each finding against actual code/plan and project conventions
- **Minimal changes only** — fix/revise what Codex flagged, nothing more. No drive-by refactoring
- **Respect project rules** — if Codex suggests something that contradicts CLAUDE.md or .claude/rules/, skip it and note why
- Always use read-only sandbox (`-s read-only`) — Codex reads, Claude fixes
- Max 5 rounds to prevent infinite loops
- If Codex CLI is not installed or fails, inform the user and suggest `npm install -g @openai/codex`
- Code fixes are left as unstaged changes — the user decides when to commit
- If a revision contradicts the user's explicit requirements, skip it and note for the user
