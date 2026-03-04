# codex-feedback-loop

Autonomous review loop for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

Codex reviews your **code changes** or **implementation plan** → Claude analyzes the feedback → Claude fixes code / revises plan → re-submits to Codex → repeats until approved. You observe the summary; Claude drives the loop.

## The Problem

Existing Codex ↔ Claude integrations are **one-shot**: Codex reviews, you read the report, you fix manually, you re-run Codex. This manual relay loop is the bottleneck — not the review itself.

**codex-feedback-loop** closes the loop:

```
You invoke /codex-feedback-loop
  ↓
Auto-detect: git diff → Code Mode | plan in context → Plan Mode
  ↓
Claude captures changes/plan → sends to Codex
  ↓
Codex returns findings (VERDICT: REVISE)
  ↓
Claude analyzes each finding:
  • Accept → fix the code / revise the plan
  • Skip → document why (false positive, project convention, etc.)
  ↓
Claude re-captures state → re-submits to Codex
  ↓
Repeat until VERDICT: APPROVED or max 5 rounds
  ↓
You see: what was fixed/revised, what was skipped, final status
```

## What Makes This Different

| Feature | codex-feedback-loop | One-shot review tools |
|---------|--------------|----------------------|
| Multi-round iteration | Up to 5 rounds with session persistence | Single pass |
| Dual mode | Auto-detects code vs plan review | Code only or plan only |
| Autonomous fixing | Claude edits source files / revises plans between rounds | Manual fixes required |
| Critical analysis | Claude triages findings (accept/skip with reasoning) | Raw report dump |
| Project-aware | Respects CLAUDE.md / .claude/rules/ conventions | Generic advice |
| Diff-aware | Auto-detects branch vs main, includes uncommitted | Manual scope |

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI)
- [OpenAI Codex CLI](https://github.com/openai/codex) v0.1.0+ (`npm install -g @openai/codex`)
  - Requires: `codex exec`, `codex exec resume`, `-s read-only`, `-o <file>` flags
- Authenticated Codex session (`codex login`)

## Installation

```bash
# Clone into Claude Code skills directory
git clone https://github.com/Alx-707/codex-feedback-loop.git ~/.claude/skills/codex-feedback-loop

# Or copy manually
cp -r codex-feedback-loop ~/.claude/skills/
```

Restart Claude Code. The `/codex-feedback-loop` skill will be available.

### Development testing

```bash
claude --plugin-dir ./codex-feedback-loop
```

## Usage

```bash
# Auto-detect mode: has diff → code review, no diff → plan review
/codex-feedback-loop

# Force plan review (even if there are code changes)
/codex-feedback-loop plan

# Use a different Codex model
/codex-feedback-loop o4-mini

# Review only specific paths (code mode)
/codex-feedback-loop -- src/lib/ src/app/api/
```

## How It Works

### Mode Detection

The skill auto-detects what to review:

1. **`plan` argument** → Plan Mode (explicit override)
2. **`git diff` has changes** → Code Mode (most common)
3. **No diff, but plan in conversation** → Plan Mode
4. **Neither** → Nothing to review

### Code Mode

1. **Capture** — Detects your branch, captures `git diff` against the base branch (+ any uncommitted changes)
2. **Review** — Sends the diff to Codex with a structured adversarial review prompt covering 5 perspectives:
   - **Correctness** — Logic errors, edge cases, wrong assumptions
   - **Security** — Input validation, injection, auth, data exposure
   - **Consistency** — Config drift, dead code, stale references
   - **Performance** — N+1, memory leaks, missing cleanup
   - **Maintainability** — Naming, complexity, error handling, test gaps
3. **Analyze** — Claude reads Codex's findings and independently triages each one:
   - **Accept** — Valid issue, will fix
   - **Skip** — False positive, style preference, or contradicts project rules (with documented reason)
4. **Fix** — Claude edits the actual source files using minimal, targeted changes
5. **Re-submit** — Captures the new diff, resumes the Codex session with context of what was fixed
6. **Repeat** — Steps 3-5 loop until `VERDICT: APPROVED` or 5 rounds

### Plan Mode

1. **Capture** — Extracts the implementation plan from the current conversation context
2. **Review** — Sends the plan to Codex for evaluation on correctness, risks, missing steps, alternatives, and security
3. **Revise** — Claude addresses each issue Codex raised, rewrites the plan
4. **Re-submit** — Resumes the Codex session with the revised plan
5. **Repeat** — Steps 3-4 loop until `VERDICT: APPROVED` or 5 rounds

### What Claude does NOT do

- Blindly apply all Codex suggestions
- Refactor surrounding code or add unrelated improvements
- Override project conventions defined in CLAUDE.md
- Commit changes — fixes are left unstaged for your review

### Session persistence

Rounds 2+ use `codex exec resume` with the captured session ID, so Codex has full context of prior reviews. If a session expires, Claude falls back to a fresh call with round history in the prompt.

## Review Perspectives (Code Mode)

The adversarial review prompt asks Codex to examine changes from 5 angles, with structured output per finding:

```
- File + line: exact location
- Severity: critical / high / medium / low
- Category: which perspective
- Issue: what's wrong
- Fix: specific suggested fix
```

Formatting/style issues are explicitly excluded — those belong to linters.

## Configuration

### Model override

Default: `gpt-5.2` with `--reasoning-effort high`. Override per invocation:

```bash
/codex-feedback-loop o4-mini
```

### Max rounds

Hardcoded at 5 to prevent infinite loops. Edit `skills/codex-feedback-loop/SKILL.md` to change.

### File filters (Code Mode)

Narrow the review scope:

```bash
/codex-feedback-loop -- src/lib/security/ src/app/api/contact/
```

## Inspiration & Prior Art

This project builds on ideas from several open-source projects:

- **[codex-plan-review](https://gist.github.com/LuD1161/84102959a9375961ad9252e4d16ed592)** by [@LuD1161](https://github.com/LuD1161) — The iterative VERDICT loop, session ID management, and `codex exec resume` pattern. codex-feedback-loop extends this from plan review to a unified code + plan review with autonomous fixing.

- **[codex-review](https://github.com/astomodynamics/codex-review)** by [@astomodynamics](https://github.com/astomodynamics) — Multi-command architecture (`/codex-bugs`, `/codex-security`, `/codex-architect`), proactive agent patterns, and the delegation skill framework. Informed the structured multi-perspective review prompt.

- **[claude-codex-review](https://github.com/pauhu/claude-codex-review)** by [@pauhu](https://github.com/pauhu) — Adversarial review from multiple viewpoints (security, correctness, compliance, performance, maintainability) and the 5-phase workflow (scope → read → check → merge → report). Shaped the review perspective design.

- **[claude-delegator](https://github.com/jarrodwatts/claude-delegator)** by [@jarrodwatts](https://github.com/jarrodwatts) — Expert-level system prompts and the concept of Claude synthesizing (not transparently passing) GPT output. Influenced the triage step where Claude independently evaluates each finding.

### Key innovation

None of the above projects implement the **autonomous fix loop** — where Claude analyzes feedback, edits real source files, and re-submits without human intervention. That's the gap codex-feedback-loop fills. The unified mode detection further simplifies usage: one command, auto-detects whether to review code or a plan.

### Customizing the review prompt

The review prompt is in `skills/codex-feedback-loop/review-prompt.md`. Edit this file to:
- Add project-specific review criteria
- Change severity definitions
- Adjust which issues to ignore

## Project Structure

```
codex-feedback-loop/
├── .claude-plugin/
│   └── plugin.json                    # Plugin manifest
├── skills/
│   └── codex-feedback-loop/
│       ├── SKILL.md                   # Skill definition (unified code + plan review)
│       └── review-prompt.md           # Adversarial review prompt (code mode, customizable)
├── LICENSE                            # MIT
└── README.md
```

This is a pure prompt-based plugin — no runtime code, no dependencies, no build step. The logic lives in `SKILL.md` as structured instructions that Claude Code interprets. The review prompt is separated into `review-prompt.md` for easy customization.

## License

MIT
