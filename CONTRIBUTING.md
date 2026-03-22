# Contributing to AI Agent Security Checklist

Thanks for your interest in making AI agents safer. Here's how to contribute.

## Adding a Checklist Item

Every checklist item must be:

1. **Actionable** — Something a team can implement or verify. Not vague advice.
2. **Specific** — Describes a concrete security measure, not a general principle.
3. **Justified** — Either explains the threat it mitigates or references a real-world incident.

### Format

```markdown
- [ ] **Short title in bold** — Detailed explanation of what to do and why. Be specific about implementation.
```

### Good Example
```markdown
- [ ] **Tool call parameter validation** — Validate all parameters against a defined schema before executing any tool call. Type check, range check, and format check. Never pass raw LLM output directly to an external API.
```

### Bad Example
```markdown
- [ ] **Be secure** — Make sure your agent is secure.
```

## Suggesting a New Section

If you think an entire category of agent security is missing, open an issue first to discuss before submitting a PR.

## Reporting a Security Issue

If you've discovered a real vulnerability in a specific AI agent product or framework, please report it to the vendor first (responsible disclosure). You can then contribute a generalised checklist item here that helps others defend against the same class of attack without disclosing vendor-specific details.

## Process

1. Fork the repo
2. Create a branch (`git checkout -b add-memory-isolation-checks`)
3. Make your changes
4. Submit a PR with a clear description of what you added and why

## Code of Conduct

Be helpful, be specific, be respectful. We're all trying to make autonomous systems safer.
