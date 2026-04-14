# Stratus Reviews

A Claude Code plugin providing code quality and security review tools for GTP-Services teams.

## Installation

### Via Stratus Marketplace (recommended)

```bash
claude plugin marketplace add GTP-Services/stratus-marketplace
claude plugin install stratus-reviews
```

### Direct Install

```bash
claude plugin install GTP-Services/stratus-reviews
```

## Usage

In Claude Code, use either skill:

```
/stratus-reviews:review-code        # Code quality review
/stratus-reviews:security-check     # Security review
```

Both skills:
- Spawn a fresh reviewer agent for isolated analysis
- Report findings in plain language
- Auto-fix critical issues
- Offer Platform team escalation for anything you cannot resolve
