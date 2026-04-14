---
name: review-code
description: Run a code quality review on your project. Spawns a code-reviewer agent, translates findings to plain language, auto-fixes critical issues, and presents remaining findings with an option to escalate to the Platform team.
triggers: [review code, check my code, code review, quality check]
---

<skill>

<purpose>
Run code quality review by spawning a fresh code-reviewer agent, process results for a non-technical user. Auto-fix critical issues (MUST-FIX), present remaining findings with clear options. Never present raw technical output — always translate to plain language.
</purpose>

<workflow>

<step name="prepare">
Verify there are changes to review:

```bash
git diff --name-only HEAD~1
```

If that returns nothing, check for uncommitted changes:

```bash
git status --short
```

If both return nothing, inform the user: "There's nothing new to review — no recent changes were found in this project." Stop here.

If changes exist, proceed.
</step>

<step name="spawn-reviewer">
Spawn a fresh code-reviewer agent using the Agent tool. Do NOT review code in the current conversation context — always delegate to a fresh agent.

Read the full agent definition from `agents/code-reviewer.md` in this plugin, then spawn:

```
Agent({
  description: "Code review for internal tool",
  subagent_type: "general-purpose",
  prompt: "[Full code-reviewer agent definition from agents/code-reviewer.md] Review the latest commit. Follow the agent definition exactly. Return findings in the specified output format.",
  model: "sonnet"
})
```

Wait for the agent to complete. Capture the full output — this is the raw review result used in the next step.
</step>

<step name="parse-results">
Parse the agent's structured output into three lists:

- **MUST-FIX** — any finding categorised as MUST-FIX (P1 or P2 severity)
- **SHOULD-FIX** — any finding categorised as SHOULD-FIX (P3 severity)
- **NOTE** — informational observations only

Extract the Verdict from the Summary section:
- If Verdict is `PASS` (zero MUST-FIX items): skip the auto-fix-loop step entirely and go directly to present-remaining.
- If Verdict is `FAIL`: proceed to auto-fix-loop.

Track counts for the final summary:
- `fixed_count` = 0
- `skipped_count` = 0
- `tickets_created` = 0
- `notes_count` = count of NOTE items
</step>

<step name="auto-fix-loop">
For each MUST-FIX item, attempt to apply the fix automatically. Run up to 3 full iterations of the fix-and-re-review cycle.

**For each MUST-FIX item:**
1. Read the file at the referenced line (from the `file:line` field in the finding)
2. Apply the concrete fix described in the "How to fix" field
3. Stage the fixed file: `git add <file>`
4. Commit: `git commit -m "fix: <plain-language description of what was fixed>"`
5. Increment `fixed_count`

**After all MUST-FIX items in the current round are addressed, re-run the review agent** (same spawn pattern as spawn-reviewer step) to verify no MUST-FIX items remain.

**Iteration limit:** Maximum 3 full fix-and-re-review cycles. If MUST-FIX items persist after 3 rounds, stop auto-fixing and present each remaining MUST-FIX to the user via AskUserQuestion:

```
"I tried to fix '[plain-language description]' automatically but couldn't get it right. What would you like to do?"
Options:
- "Try to fix it differently"   → attempt an alternative approach and re-run (does not count as a new iteration)
- "Skip this for now"           → increment skipped_count, continue
- "Open a Platform ticket for me" → invoke request-platform-help skill with the finding details, increment tickets_created
```

Never present the raw technical output. Translate every finding to plain language before asking the user.
</step>

<step name="present-remaining">
For each SHOULD-FIX item, present it to the user one at a time via AskUserQuestion. Translate the finding to plain language before presenting.

Example format:
```
"[Plain-language description of what could be better and why it matters]. Would you like to address this?"
Options:
- "Fix it for me"                            → apply the suggested fix, stage and commit, increment fixed_count
- "Skip this"                                → increment skipped_count, continue to next
- "I'm not sure — open a Platform ticket for me" → invoke request-platform-help skill with the finding details, increment tickets_created
```

Process all SHOULD-FIX items before moving to the summary step. NOTE items do not require user decisions — collect them for the summary only.
</step>

<step name="summary">
Present the final review summary in plain language:

```
Code review complete:
- X issues found and fixed automatically
- X issues you chose to skip
- X Platform tickets created
- X observations (no action needed)
```

Then add a verdict line:
- If all MUST-FIX items were resolved (either fixed or escalated via ticket): "Your code passed the quality check."
- If any MUST-FIX items remain unresolved (skipped by user): "Some issues remain — the Platform team will help with the tickets created." or if no tickets were created: "Some issues were skipped. Consider revisiting them before shipping."
</step>

</workflow>

<constraints>
- MUST spawn a FRESH agent — never review code in the current conversation context
- MUST auto-fix MUST-FIX items before presenting them to the user
- MUST use AskUserQuestion for every user-facing decision
- MUST include "Open a Platform ticket" as an option for every non-auto-fixable issue
- Maximum 3 auto-fix iterations before escalating remaining MUST-FIX items to the user
- NEVER present raw technical output — always translate to plain language before presenting to the user
- NEVER skip the spawn-reviewer step — reviewing in the current context defeats the purpose of isolation
- When invoking request-platform-help, pass the raw finding text and plain-language translation so the skill can compose the ticket without asking the user to repeat themselves
</constraints>

</skill>
