# ToneShift MVP Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Product:** ToneShift - AI Tone Converter for Any Text

**Tagline:** "Shift your tone in 1 click"

**Goal:** Build a secure, freemium tone converter với web app và browser extensions (Chrome, Firefox, Edge). Extension hoạt động trên MỌI text input trên web, không chỉ email.

**Architecture:** Monorepo với React frontend (Vite), Node.js/Express backend, Firebase Auth + Firestore cho user management và quota tracking, Stripe cho payments. Tất cả API calls đều đi qua authenticated backend để ngăn quota bypass.

**Tech Stack:** React 18, TypeScript, Tailwind CSS, Vite, Node.js, Express, Firebase Auth, Firestore, OpenAI API (gpt-4o-mini), Stripe, Vercel, Browser Extensions (Chrome MV3, Firefox, Edge)

**Lưu ý commits:** KHÔNG ghi Co-Author trong bất kỳ commit nào.

---

## Product Overview

**ToneShift** giúp users chuyển đổi văn bản sang 8 tone khác nhau, hoạt động trên:
- ✅ Mọi text input/textarea trên web
- ✅ Gmail, Outlook web
- ✅ Social media (LinkedIn, Twitter, Facebook)
- ✅ Slack, Discord web
- ✅ CMS editors (WordPress, Notion)
- ✅ Any contenteditable element

| Tone | Use Case |
|------|----------|
| Formal | Business emails, reports, proposals |
| Casual | Social media, chats, forums |
| Professional | LinkedIn, cover letters, client communication |
| Persuasive | Sales, marketing, pitches |
| Friendly | Customer support, team chats |
| Enthusiastic | Announcements, promotions |
| Empathetic | Apologies, sensitive topics |
| Direct | Instructions, urgent requests |

---

## Pricing Review

### Phân tích thị trường
| Competitor | Price | Features |
|------------|-------|----------|
| Grammarly Free | $0 | Basic grammar |
| Grammarly Premium | $12/mo | Tone (3 options) |
| Wordtune | $9.99/mo | Rewrite + 5 tones |
| QuillBot | $9.95/mo | Paraphrasing |

### Đề xuất pricing (competitive & profitable)

| Tier | Price | Users | Daily Limit | Target |
|------|-------|-------|-------------|--------|
| **FREE** | $0 | 1 | 10/day | Casual users, trial |
| **PRO** | $4.99/mo | 1 | Unlimited | Individual power users |
| **TEAM** | $13.99/mo flat | Up to 5 | Unlimited | Small teams (Phase 2) |
| **ENTERPRISE** | Custom | Unlimited | Unlimited + API | Large orgs (Phase 3) |

**Lý do:**
- Free: 10/day generous để hook users
- Pro: $4.99/mo = ~125k VND (vẫn rẻ hơn Grammarly $12, Wordtune $10)
- Team: $13.99 cho 5 users = **$2.80/user** (rẻ hơn Pro 44%!)
- Margin an toàn, competitive pricing

---

## Cost Analysis (OpenAI GPT-4o-mini)

### API Pricing
| Token Type | Cost |
|------------|------|
| Input | $0.15 / 1M tokens |
| Output | $0.60 / 1M tokens |

### Cost per Conversion
```
Input:  500 tokens × $0.00000015 = $0.000075
Output: 400 tokens × $0.0000006  = $0.00024
────────────────────────────────────────────
Total: ~$0.0003 → round up to $0.001/conversion
```

### Profit Margin Analysis
| Tier | Price | Scenario | Conv/mo | Cost | Profit | Margin |
|------|-------|----------|---------|------|--------|--------|
| FREE | $0 | Active (10/day) | 300 | $0.30 | -$0.30 | Loss (CAC) |
| PRO | $4.99 | Heavy (50/day) | 1,500 | $1.50 | $3.49 | 70% ✅ |
| PRO | $4.99 | Average (20/day) | 600 | $0.60 | $4.39 | 88% ✅ |
| TEAM | $13.99 | 5 heavy | 7,500 | $7.50 | $6.49 | 46% ✅ |
| TEAM | $13.99 | 5 average | 3,000 | $3.00 | $10.99 | 79% ✅ |

**Pricing strategy:**
- Margin healthy ở mọi scenario (min 46%)
- Vẫn rẻ hơn competitors (Grammarly $12, Wordtune $10)
- Team discount hấp dẫn: $2.80/user vs $4.99 (44% savings)

---

## Infrastructure Costs

### Service Pricing

| Service | Free Tier | Paid Plan |
|---------|-----------|-----------|
| **Vercel** | 150K functions (personal only) | $20/mo Pro |
| **Firebase Auth** | 50K MAU | $0.0055/MAU |
| **Firestore** | 50K reads, 20K writes/day | $0.18/100K ops |
| **Stripe** | - | 2.9% + $0.30/transaction |
| **Chrome Store** | - | $5 one-time |
| **Firefox/Edge** | Free | Free |
| **Domain** | - | ~$12/year |

### One-time Startup Cost

| Item | Cost |
|------|------|
| Chrome Web Store | $5 |
| Domain (.io) | $12 |
| **Total** | **$17** |

### Monthly Cost by Scale

| Scale | Users | Paid | Vercel | Firebase | OpenAI | Stripe | Total | Revenue | Margin |
|-------|-------|------|--------|----------|--------|--------|-------|---------|--------|
| Launch | 1K | 100 | $20 | $0 | $30 | $15 | **$65** | $499 | **87%** |
| Growth | 10K | 1K | $35 | $20 | $200 | $150 | **$405** | $4,990 | **92%** |
| Scale | 100K | 10K | $150 | $200 | $1,500 | $1,500 | **$3,350** | $49,900 | **93%** |

### Cost Breakdown (at scale)

```
OpenAI API:    ~45% of costs (largest)
Stripe fees:   ~45% of costs
Infra (Vercel/Firebase): ~10% of costs
```

**Kết luận:** Business model sustainable với margin 87-93% ở mọi scale.

---

## Security Architecture

### Vấn đề 1: Users bypass quota bằng cách gọi API trực tiếp
**Giải pháp:** Server-side quota enforcement
- Tất cả OpenAI calls chỉ đi qua backend (không bao giờ từ client)
- Backend validate Firebase Auth token trên mọi request
- Quota tracked theo `userId` trong Firestore với atomic transactions
- Rate limiting ở API gateway level (Express middleware)

### Vấn đề 2: Users xóa cài lại extension/clear data để reset quota
**Giải pháp:** Account-based identity
- Quota gắn với Firebase Auth `userId`, không phải device/extension
- Bắt buộc email verification trước khi convert lần đầu
- Không cho anonymous usage - phải sign in
- Quota lưu trong Firestore (server-side), không phải localStorage

### Vấn đề 3: Direct API abuse
**Giải pháp:** Defense in depth
- CORS chỉ cho phép origins đã whitelist
- API key chỉ lưu ở backend environment (không expose ra client)
- IP-based rate limiting làm layer bảo vệ thêm

---

## Development Phases

### Phase 1: MVP (Week 1-2) ← CURRENT PLAN
- Web app
- Chrome extension (all text inputs)
- Core backend với security

### Phase 2: Multi-Browser (Week 3-4)
- Firefox extension (WebExtensions API compatible)
- Edge extension (Chromium-based, minimal changes)
- Team tier với shared billing

### Phase 3: Integrations (Month 2)
- API for developers
- Zapier integration
- Slack app
- Enterprise features

---

## Task Overview - Phase 1 (17 Tasks)

### Phase 1.1: Project Setup (Tasks 1-2)
| Task | Mô tả | Files |
|------|-------|-------|
| 1 | Initialize monorepo với pnpm workspaces | `package.json`, `pnpm-workspace.yaml`, `.gitignore`, `.nvmrc` |
| 2 | Setup backend project với Express + TypeScript | `apps/backend/*` |

### Phase 1.2: Backend Security Layer (Tasks 3-6)
| Task | Mô tả | Files |
|------|-------|-------|
| 3 | Firebase Admin SDK setup | `apps/backend/src/config/firebase.ts` |
| 4 | Authentication middleware (require email verified) | `apps/backend/src/middleware/auth.ts` |
| 5 | **Quota Service với atomic Firestore transactions** | `apps/backend/src/services/quota.ts` |
| 6 | Rate limiting middleware (IP + user based) | `apps/backend/src/middleware/rateLimit.ts` |

### Phase 1.3: Core Features (Tasks 7-9)
| Task | Mô tả | Files |
|------|-------|-------|
| 7 | OpenAI tone conversion service (8 tones) | `apps/backend/src/services/openai.ts` |
| 8 | Conversion API endpoint với quota check | `apps/backend/src/routes/convert.ts` |
| 9 | Stripe payment integration + webhooks | `apps/backend/src/services/stripe.ts`, `apps/backend/src/routes/billing.ts` |

### Phase 1.4: Frontend (Tasks 10-13)
| Task | Mô tả | Files |
|------|-------|-------|
| 10 | Frontend setup với Vite + React + Tailwind | `apps/web/*` |
| 11 | Firebase Auth integration (Google sign-in) | `apps/web/src/lib/firebase.ts`, `apps/web/src/stores/authStore.ts` |
| 12 | API client với token management | `apps/web/src/lib/api.ts` |
| 13 | Main ToneShift UI với quota display | `apps/web/src/components/ToneShift.tsx` |

### Phase 1.5: Chrome Extension (Task 14)
| Task | Mô tả | Files |
|------|-------|-------|
| 14 | Chrome extension cho ALL text inputs | `apps/extension/chrome/*` |

### Phase 1.6: Build & Deploy (Tasks 15-17)
| Task | Mô tả | Files |
|------|-------|-------|
| 15 | Build configuration cho all apps | `package.json`, `apps/*/vite.config.ts` |
| 16 | Vercel deployment configuration | `apps/*/vercel.json` |
| 17 | Security documentation | `docs/SECURITY.md` |

---

## Task Overview - Phase 2: Multi-Browser (4 Tasks)

| Task | Mô tả | Files |
|------|-------|-------|
| 18 | Shared extension core (browser-agnostic) | `packages/extension-core/*` |
| 19 | Firefox extension adaptation | `apps/extension/firefox/*` |
| 20 | Edge extension adaptation | `apps/extension/edge/*` |
| 21 | Team billing & shared workspace | `apps/backend/src/services/team.ts` |

---

## Chrome Extension - Universal Text Input Support

### Content Script Strategy

```typescript
// Detect ALL text input types on any webpage
const TEXT_INPUT_SELECTORS = [
  'textarea',                           // Standard textarea
  'input[type="text"]',                 // Text inputs
  'input[type="email"]',                // Email inputs
  'input[type="search"]',               // Search inputs
  '[contenteditable="true"]',           // Rich text editors
  '[role="textbox"]',                   // ARIA textboxes (Gmail, Slack)
  '.ql-editor',                         // Quill editor
  '.ProseMirror',                       // ProseMirror (Notion, many CMS)
  '.tox-edit-area__iframe',             // TinyMCE
  '.cke_editable',                      // CKEditor
];

// Inject ToneShift button near focused input
function injectToneShiftButton(element: HTMLElement) {
  // Position button near the input
  // Show tone dropdown on click
  // Replace text with converted result
}
```

### Supported Platforms (auto-detected)
- Gmail, Outlook.com, Yahoo Mail
- LinkedIn, Twitter/X, Facebook, Instagram
- Slack, Discord, Microsoft Teams (web)
- Notion, WordPress, Medium
- GitHub, GitLab (comments, issues)
- Reddit, Quora, forums
- Any website with text inputs

---

## Browser Extension Compatibility

### Shared Core (packages/extension-core)
```
packages/extension-core/
├── src/
│   ├── content/
│   │   ├── detector.ts      # Detect text inputs
│   │   ├── injector.ts      # Inject UI
│   │   └── converter.ts     # Call API
│   ├── popup/
│   │   └── App.tsx          # Popup UI (React)
│   ├── background/
│   │   └── service.ts       # Auth token management
│   └── styles/
│       └── toneshift.css    # Injected styles
└── package.json
```

### Platform-Specific Adapters

| Browser | Manifest | API Differences |
|---------|----------|-----------------|
| Chrome | Manifest V3 | `chrome.` namespace, service worker |
| Firefox | Manifest V2/V3 | `browser.` namespace (Promise-based) |
| Edge | Manifest V3 | Same as Chrome (Chromium-based) |

### Build Strategy
```bash
# Single source, multiple outputs
pnpm build:chrome    # → apps/extension/chrome/dist
pnpm build:firefox   # → apps/extension/firefox/dist
pnpm build:edge      # → apps/extension/edge/dist
```

---

## Updated Project Structure

```
toneshift/
├── apps/
│   ├── backend/              # Express API server
│   │   ├── src/
│   │   │   ├── config/       # Firebase config
│   │   │   ├── middleware/   # Auth, rate limiting
│   │   │   ├── services/     # Quota, OpenAI, Stripe
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

---

## Chi tiết Security Implementation

### Task 5: Quota Service (Critical)

```typescript
// Atomic check-and-increment để ngăn race conditions
static async checkAndIncrementQuota(userId: string): Promise<{
  allowed: boolean;
  remaining: number;
  tier: UserTier;
}> {
  const db = firestore();
  const userRef = db.collection('users').doc(userId);

  return db.runTransaction(async (transaction) => {
    const userDoc = await transaction.get(userRef);
    const userData = userDoc.data();

    // Check quota
    if (userData.dailyUsage >= LIMITS[userData.tier]) {
      return { allowed: false, remaining: 0, tier: userData.tier };
    }

    // Increment atomically
    transaction.update(userRef, {
      dailyUsage: FieldValue.increment(1),
      lastResetDate: today,
    });

    return { allowed: true, remaining: limit - newUsage, tier };
  });
}
```

### Updated Quota Limits

```typescript
const DAILY_LIMITS: Record<UserTier, number> = {
  free: 10,        // Generous free tier
  pro: -1,         // Unlimited
  team: -1,        // Unlimited, up to 5 users (Phase 2)
  enterprise: -1,  // Unlimited users (Phase 3)
};

const MAX_TEAM_MEMBERS = 5; // Team tier limit
```

---

## Tóm tắt Security Measures

| Threat | Mitigation |
|--------|------------|
| Quota bypass qua direct API | Backend-only conversion, token required |
| Quota bypass qua multiple accounts | Email verification bắt buộc |
| Quota bypass qua reinstall | Quota gắn với Firebase user ID |
| Rate limit bypass | IP + user-based rate limiting |
| API key exposure | Keys chỉ lưu server-side |
| Race condition quota bypass | Atomic Firestore transactions |

---

## Branding Guidelines

**Name:** ToneShift

**Tagline:** "Shift your tone in 1 click"

**Color palette:**
- Primary: #6366F1 (Indigo - modern, creative)
- Secondary: #10B981 (Green - success)
- Accent: #F59E0B (Orange - CTA)

**Logo concept:** Letter "T" với wave/shift effect

---

## Execution Checklist - Phase 1

- [ ] Task 1: Project Initialization
- [ ] Task 2: Backend Project Setup
- [ ] Task 3: Firebase Admin SDK Setup
- [ ] Task 4: Authentication Middleware (email verification)
- [ ] Task 5: Quota Service (atomic transactions) ⚠️ CRITICAL
- [ ] Task 6: Rate Limiting Middleware
- [ ] Task 7: OpenAI Tone Conversion Service
- [ ] Task 8: Conversion API Endpoint
- [ ] Task 9: Stripe Payment Integration
- [ ] Task 10: Frontend Project Setup
- [ ] Task 11: Firebase Auth Integration
- [ ] Task 12: API Client
- [ ] Task 13: Main ToneShift UI
- [ ] Task 14: Chrome Extension (universal text input)
- [ ] Task 15: Build Configuration
- [ ] Task 16: Deployment Configuration
- [ ] Task 17: Security Documentation

## Execution Checklist - Phase 2

- [ ] Task 18: Shared Extension Core
- [ ] Task 19: Firefox Extension
- [ ] Task 20: Edge Extension
- [ ] Task 21: Team Billing

---

## Summary: Key Changes từ Plan Cũ

| Aspect | Before | After |
|--------|--------|-------|
| Scope | Email only (Gmail) | ALL text inputs on web |
| Free tier | 5/day | 10/day |
| Pricing | $4.99 Starter, $29.99 Pro | $4.99 Pro, $13.99 Team (5 users) |
| Tiers | 4 tiers complex | 2 tiers MVP (FREE, PRO) → TEAM Phase 2 |
| Browsers | Chrome only | Chrome (MVP) + Firefox, Edge (Phase 2) |
| Extension architecture | Single | Shared core + adapters |

## Pricing Logic

```
FREE     → $0       → 10/day     → Hook users
PRO      → $4.99    → Unlimited  → $4.99/user
TEAM     → $13.99   → 5 users    → $2.80/user (44% savings!)
```

→ Clear upgrade path: dùng nhiều người = tiết kiệm hơn
→ Healthy margins: min 46% even worst case

---

**Plan đã hoàn thành. Chờ bạn cho phép để bắt đầu implement.**
