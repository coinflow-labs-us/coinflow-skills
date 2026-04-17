# Coinflow AI Skills

AI skills for integrating [Coinflow](https://coinflow.cash) payments, payouts, webhooks, and subscriptions into your application.

These are structured instructions that you install into **your own AI coding tool** — Claude Code, Cursor, VS Code Copilot, Windsurf, and [16+ other tools](https://github.com/vercel-labs/skills) — to get Coinflow-aware code generation, debugging, and implementation guidance.

Skills run **entirely locally**. They do not transmit your code to Coinflow or any third party.

## What's included

| Skill | Description |
|-------|-------------|
| **[Checkout Integration](./checkout-integration/)** | Payment flows for cards, ACH, Apple Pay, Google Pay, SEPA, PIX, USDC, and crypto. Covers React SDK, API, checkout links, 3DS challenges, card tokenization, saved cards, Card on File, zero authorization, and chargeback protection. |
| **[Webhook Integration](./webhook-integration/)** | Webhook setup, signature verification, event handling, deduplication, and payload structures for all checkout, subscription, payout, and KYC events. |
| **[Payout Integration](./payout-integration/)** | KYC/KYB flows, payout method setup (bank, debit card, Venmo, PayPal, Interac, IBAN, PIX), quotes, and payout initiation across all regions and speeds. |
| **[API Integration](./api-integration/)** | Authentication, session keys, environment setup, rate limits, SDK configuration, subscriptions, marketplaces, and cross-cutting concerns. |

## Installation

```bash
npx skills add coinflow-labs/coinflow-skills
```

Or install specific skills:
```bash
npx skills add coinflow-labs/coinflow-skills --skill checkout-integration
```

## Resources

- [Coinflow Documentation](https://docs.coinflow.cash)
- [API Reference](https://docs.coinflow.cash/api-reference)
- [Sandbox Dashboard](https://sandbox-merchant.coinflow.cash)
