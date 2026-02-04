# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**ToneShift** - AI Tone Converter for Any Text

Tagline: "Shift your tone in 1 click"

A browser extension + web app that converts any text to 8 different tones (Formal, Casual, Professional, Persuasive, Friendly, Enthusiastic, Empathetic, Direct). Works on ALL text inputs across the web - not just email.

**Current Status:** Pre-development. Business plan in `complete_package.md`, implementation plan in `docs/plans/`.

## Tech Stack (Planned)

- **Frontend:** React 18 + TypeScript, Tailwind CSS, Vite, Zustand (state management)
- **Chrome Extension:** Manifest V3
- **Backend:** Node.js + Express, REST API
- **Database:** Google Firestore
- **Auth:** Firebase Authentication
- **AI:** OpenAI API (gpt-4o-mini)
- **Payments:** Stripe
- **Hosting:** Vercel (frontend and backend)

## Architecture

```
toneshift/
├── web-app/              # React web application
│   ├── src/
│   │   ├── components/   # UI components
│   │   ├── hooks/        # Custom React hooks
│   │   ├── store/        # Zustand state management
│   │   └── services/     # API client services
├── chrome-extension/     # Manifest V3 extension
│   ├── manifest.json
│   ├── content/          # Content scripts for Gmail
│   ├── popup/            # Extension popup UI
│   └── background/       # Service worker
├── backend/              # Express API server
│   ├── src/
│   │   ├── routes/       # API routes
│   │   ├── services/     # Business logic (OpenAI, Stripe)
│   │   ├── middleware/   # Auth, rate limiting
│   │   └── config/       # Firebase, OpenAI config
└── shared/               # Shared types/utilities
```

## Core Features (MVP)

1. **ToneShift Engine:** 8 tones via OpenAI API (gpt-4o-mini)
2. **Web App:** Paste email → select tone → get converted result
3. **Chrome Extension:** Gmail integration with inline ToneShift button
4. **User Auth:** Firebase Authentication (email verification required)
5. **Quota System:** Free tier (5/day), Paid tier (unlimited) - server-side enforcement
6. **Payments:** Stripe subscriptions ($4.99/month starter)

## Key Integration Points

- **Gmail:** Content script injects tone conversion UI into compose window
- **OpenAI:** Backend proxies requests to protect API key
- **Firebase:** Auth tokens validated on backend, Firestore stores user data and usage quotas
- **Stripe:** Webhook handlers for subscription events

## Business Rules

- Free tier: 5 conversions per day per user
- Starter ($4.99/mo): 50 conversions per day
- Professional ($29.99/mo): Unlimited conversions
- API cost: ~$0.02 per conversion (gpt-4o-mini)
