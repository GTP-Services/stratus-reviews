---
name: security-check
description: Run a security review on your project. Spawns a security-reviewer agent, translates findings to business impact language, auto-fixes critical issues, and presents remaining findings with Platform team escalation option.
triggers: [security check, security review, is my code secure, check for vulnerabilities]
---

<skill>

<purpose>
Run security review by spawning a fresh security-reviewer agent, process results for a non-technical user. Auto-fix CRITICAL and HIGH (P1/P2) findings before presenting anything to the user. Present remaining MEDIUM findings with clear options including Platform team escalation. Never present raw technical output — always translate to business impact language. Never display actual secret values found during review.
</purpose>

<workflow>

<step name="prepare">
Identify the scope of the review.

If invoked by the orchestrator: review the full project.

If invoked independently, check for recent changes:

```bash
git diff --name-only HEAD~1
```

If that returns files, review those changes. If it returns nothing, offer the user a choice via AskUserQuestion:

```
"There are no recent changes to your project. What would you like me to review?"
Options:
- "Review the full project"  → proceed with full project scan
- "Never mind"               → stop here
```
</step>

<step name="spawn-reviewer">
Spawn a fresh security-reviewer agent using the Agent tool. Do NOT perform the security review in the current conversation context — always delegate to a fresh agent.

Read the full agent definition from `agents/security-reviewer.md` in this plugin, then spawn:

```
Agent({
  description: "Security review for internal tool",
  subagent_type: "general-purpose",
  prompt: "[Full security-reviewer agent definition from agents/security-reviewer.md] Review this repository for security vulnerabilities. Follow the agent definition exactly. Return findings in the specified output format.",
  model: "sonnet"
})
```

Wait for the agent to complete. Capture the full output — this is the raw review result used in the next step.
</step>

<step name="parse-results">
Parse the agent's structured output into severity groups:

- **CRITICAL + HIGH** (P1/P2) — auto-fix candidates
- **MEDIUM** (P3) — present to user for decision
- **LOW + INFO** — informational, include in summary only

Extract the Verdict from the Summary section:
- If Verdict is `SECURE` (no CRITICAL or HIGH findings): skip the auto-fix-loop step and go directly to present-remaining.
- If Verdict is `CONCERNS` or `INSECURE`: proceed to auto-fix-loop.

Track counts for the final summary:
- `auto_fixed_count` = 0
- `medium_fixed_count` = 0
- `medium_skipped_count` = 0
- `tickets_created` = 0
- `low_info_count` = count of LOW + INFO items
</step>

<step name="auto-fix-loop">
For each CRITICAL and HIGH finding, attempt to apply the fix automatically. Run up to 3 full iterations of the fix-and-re-review cycle.

**For each CRITICAL or HIGH finding:**
1. Read the file at the referenced line (from the `file:line` field in the finding)
2. Apply the concrete fix described in the "How to fix" field
3. Stage the fixed file: `git add <file>`
4. Commit: `git commit -m "security: fix <plain-language description of what was fixed>"`
5. Increment `auto_fixed_count`

**IMPORTANT:** Never display actual secret values found during review. If a secret is embedded in code, replace it with a placeholder and note that the real value must come from an environment variable or secret manager.

**After all CRITICAL/HIGH items in the current round are addressed, re-run the security agent** (same spawn pattern as spawn-reviewer step) to verify no CRITICAL or HIGH findings remain.

**Iteration limit:** Maximum 3 full fix-and-re-review cycles. If CRITICAL or HIGH findings persist after 3 rounds, stop auto-fixing and present each remaining item to the user via AskUserQuestion:

```
"I tried to fix '[plain-language description of the security issue and what could go wrong]' automatically but couldn't resolve it. What would you like to do?"
Options:
- "Try a different fix"                   → attempt an alternative approach and re-run (does not count as a new iteration)
- "Skip this for now"                     → see CRITICAL skip confirmation below
- "Open a Platform ticket for me"         → invoke request-platform-help skill with the finding details, increment tickets_created
```

**Extra confirmation for skipping CRITICAL findings.** If the user chooses "Skip this for now" on a CRITICAL finding, present an additional confirmation via AskUserQuestion before proceeding:

```
"This is a serious security issue. Are you sure you want to ship without fixing it? The Platform team can usually resolve these quickly."
Options:
- "Yes, skip it — I'll accept the risk"    → increment medium_skipped_count, continue
- "Actually, open a Platform ticket for me" → invoke request-platform-help skill with the finding details, increment tickets_created
```

Never present raw technical output. Translate every finding to plain business-impact language before asking the user.
</step>

<step name="present-remaining">
For each MEDIUM finding, present it to the user one at a time via AskUserQuestion. Translate the finding to plain language with business impact framing before presenting.

Example format:
```
"[Plain-language description of the issue and what could go wrong for users or the business]. Would you like to address this?"
Options:
- "Fix it for me"                                    → apply the suggested fix, stage and commit with message "security: fix <description>", increment medium_fixed_count
- "Skip this"                                        → acceptable risk for an internal tool, increment medium_skipped_count, continue
- "I'm not sure — open a Platform ticket for me"     → invoke request-platform-help skill with the finding details, increment tickets_created
```

Process all MEDIUM items before moving to the summary step. LOW and INFO items do not require user decisions — collect them for the summary only.
</step>

<step name="summary">
Present the final security review summary:

```
Security review complete:
- Verdict: [SECURE / CONCERNS / INSECURE]
- X critical/high issues found and fixed automatically
- X medium issues [fixed/skipped/escalated]
- X low/informational observations
- X Platform tickets created
```

Then add a verdict line based on outcome:
- If Verdict `SECURE`: "No security concerns found. Your app is ready to ship."
- If CONCERNS after all fixes complete: "All high-priority issues resolved. Remaining items are low risk."
- If INSECURE with any unresolved CRITICAL findings remaining: "WARNING: Critical security issues remain. Strongly recommend Platform team review before shipping."
</step>

</workflow>

<constraints>
- MUST spawn a FRESH agent — never review in the current conversation context
- MUST auto-fix CRITICAL and HIGH before presenting anything to the user
- MUST use AskUserQuestion for every user-facing decision
- MUST include extra confirmation before allowing the user to skip a CRITICAL finding
- MUST include "Open a Platform ticket" as an option for every finding that cannot be auto-fixed
- Maximum 3 auto-fix iterations before escalating remaining CRITICAL/HIGH items to the user
- NEVER present raw technical output — always translate to business impact language
- NEVER display actual secret values found during review
- NEVER skip the spawn-reviewer step — reviewing in current context defeats the purpose of isolation
- When invoking request-platform-help, pass the raw finding text and plain-language translation so the skill can compose the ticket without asking the user to repeat themselves
</constraints>

</skill>
