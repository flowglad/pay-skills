# Flowglad Pay Skills

Skills for working with Flowglad Pay virtual payment cards.

## Repository Structure

```text
skills/
├── card-sessions/SKILL.md       # Create and redeem card sessions
└── agent-card-setup/SKILL.md    # Agent onboarding with cards and policies
```

## When to Use These Skills

Use these skills when the user wants to:
- Get virtual card details (number, CVV, expiry) via card sessions
- Set up an AI agent with a spending-limited virtual card
- Understand the Flowglad Pay card session security model
- Diagnose why an agent can't access card details

## Skill Selection

| User Intent | Skill |
|-------------|-------|
| "Get card details" | card-sessions |
| "Create a card session" | card-sessions |
| "Redeem a card session" | card-sessions |
| "Audit card session usage" | card-sessions |
| "Set up a card for my agent" | agent-card-setup |
| "Check agent onboarding status" | agent-card-setup |
| "Attach spending policies" | agent-card-setup |
| "Why can't my agent get card details?" | agent-card-setup |

## Important Conventions

1. **Card details are never exposed through MCP** — the MCP tools intentionally omit card PAN/CVV. Use the REST API redeem endpoint with a scoped token instead.
2. **Scoped tokens, not API keys** — card session redemption uses a short-lived JWT in the `X-Scoped-Token` header, not the standard API key.
3. **IDs have prefixes** — payment methods start with `pm_`, card sessions with `cs_`, redemptions with `csr_`, Brex cards with `ncard_`.
