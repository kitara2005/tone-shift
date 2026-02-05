# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**ToneShift** - AI Tone Converter for Any Text

Tagline: "Shift your tone in 1 click"

A browser extension + web app that converts any text to 8 different tones (Formal, Casual, Professional, Persuasive, Friendly, Enthusiastic, Empathetic, Direct). Works on ALL text inputs across the web - not just email.

**Current Status:** Pre-development. Business plan in `complete_package.md`, implementation plan in `docs/plans/`.

## Tech Stack (Planned)

- **Frontend:** React 18 + TypeScript, Tailwind CSS, Vite, Zustand (state management)
- **Browser Extensions:** Chrome (MV3), Firefox, Edge
- **Backend:** Node.js + Express, REST API
- **Database:** Google Firestore
- **Auth:** Firebase Authentication (email verification required)
- **AI:** Tiered LLM System
  - Free: GPT-4.1 nano (cost-optimized, $0.00007/conv)
  - Pro/Team: Claude 3 Haiku (quality-optimized, $0.000625/conv)
  - Fallback: GPT-4o-mini (all tiers)
- **Payments:** Stripe
- **Hosting:** Vercel (frontend and backend)

## Architecture

```
toneshift/
├── apps/
│   ├── backend/              # Express API server
│   │   ├── src/
│   │   │   ├── config/       # Firebase, LLM config
│   │   │   ├── middleware/   # Auth, rate limiting
│   │   │   ├── services/     # LLM, Quota, Stripe
│   │   │   │   └── llm/      # Tiered LLM providers
│   │   │   └── routes/       # API endpoints
│   │   └── package.json
│   ├── web/                  # React web app
│   │   ├── src/
│   │   │   ├── components/   # UI components
│   │   │   ├── lib/          # Firebase, API client
│   │   │   └── stores/       # Zustand stores
│   │   └── package.json
│   └── extension/
│       ├── chrome/           # Chrome MV3
│       ├── firefox/          # Firefox (Phase 2)
│       └── edge/             # Edge (Phase 2)
├── packages/
│   └── extension-core/       # Shared extension code (Phase 2)
├── docs/
│   └── plans/
├── package.json
└── pnpm-workspace.yaml
```

## Core Features (MVP)

1. **ToneShift Engine:** 8 tones via tiered LLM system
2. **Web App:** Paste text → select tone → get converted result
3. **Chrome Extension:** Works on ALL text inputs (Gmail, LinkedIn, Slack, etc.)
4. **User Auth:** Firebase Authentication (email verification required)
5. **Quota System:** Free tier (10/day), Pro (unlimited) - server-side enforcement
6. **Payments:** Stripe subscriptions

## Pricing Model

| Tier | Price | Users | Daily Limit | LLM Model |
|------|-------|-------|-------------|-----------|
| FREE | $0 | 1 | 10/day | GPT-4.1 nano |
| PRO | $4.99/mo | 1 | Unlimited | Claude 3 Haiku |
| TEAM | $13.99/mo | Up to 5 | Unlimited | Claude 3 Haiku |

## Key Integration Points

- **Browser Extension:** Content script detects text inputs, injects ToneShift button
- **LLM Providers:** Anthropic (Claude) + OpenAI with automatic fallback
- **Firebase:** Auth tokens validated on backend, Firestore stores user data and quotas
- **Stripe:** Webhook handlers for subscription events

## Security

- All LLM calls go through backend only (API keys never exposed)
- Quota tied to Firebase user ID (prevents reinstall abuse)
- Email verification required before first conversion
- Atomic Firestore transactions for quota enforcement
- Rate limiting: IP + user-based

## Commit Guidelines

- KHÔNG ghi Co-Author trong bất kỳ commit nào
