# LLM Model Cost Optimization Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement tiered LLM infrastructure where Free users get cost-optimized models and Pro/Team users get premium quality models, with flexible configuration for A/B testing.

**Architecture:** Multi-provider abstraction layer with tier-based model selection, runtime switching via environment config, automatic fallback support, and user feedback collection.

**Tech Stack:** Node.js, Express, OpenAI SDK, Anthropic SDK, environment-based configuration, Firestore for feedback

---

## Research Summary

### Complete Model Comparison (2026)

| Model | Provider | Input ($/1M) | Output ($/1M) | Cost/Conv* | Quality | Best For |
|-------|----------|--------------|---------------|------------|---------|----------|
| **GPT-4.1 nano** | OpenAI | $0.02 | $0.15 | **$0.00007** | ⭐⭐⭐ | Free tier, high volume |
| **Gemini 1.5 Flash** | Google | $0.075 | $0.30 | $0.00016 | ⭐⭐⭐ | Speed-critical |
| **GPT-4o-mini** | OpenAI | $0.15 | $0.60 | $0.000315 | ⭐⭐⭐⭐ | Balanced |
| **Claude 3 Haiku** | Anthropic | $0.25 | $1.25 | $0.000625 | ⭐⭐⭐⭐⭐ | Writing quality |
| **GPT-4.1 mini** | OpenAI | $0.40 | $1.60 | $0.00084 | ⭐⭐⭐⭐⭐ | Premium tier |
| **Claude 3.5 Haiku** | Anthropic | $1.00 | $5.00 | $0.0025 | ⭐⭐⭐⭐⭐ | Enterprise |

*500 input + 400 output tokens

### Key Findings

1. **GPT-4.1 nano** is 4.5x cheaper than GPT-4o-mini ($0.00007 vs $0.000315)
2. **Claude 3 Haiku** has 30% fewer hallucinations than GPT models
3. **GPT-4.1 mini** matches GPT-4.1 quality at 1/5th price
4. **Tiered approach** can reduce costs by 80%+ while maintaining quality for paying users

---

## Tiered Model Strategy

### Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    TIERED MODEL ARCHITECTURE                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  FREE TIER (10/day)                                         │
│  ├── Primary: GPT-4.1 nano ($0.00007/conv)                  │
│  ├── Fallback: GPT-4o-mini                                  │
│  └── Goal: Minimize cost, collect feedback                  │
│                                                              │
│  PRO TIER ($4.99/mo, unlimited)                             │
│  ├── Primary: Claude 3 Haiku ($0.000625/conv)               │
│  ├── Fallback: GPT-4o-mini                                  │
│  └── Goal: Best writing quality                             │
│                                                              │
│  TEAM TIER ($13.99/mo, 5 users)                             │
│  ├── Primary: Claude 3 Haiku or GPT-4.1 mini                │
│  ├── Fallback: GPT-4o-mini                                  │
│  └── Goal: Premium quality + reliability                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Cost Projection by Tier

| Tier | Users | Conv/mo | Model | Cost/Conv | Monthly Cost |
|------|-------|---------|-------|-----------|--------------|
| FREE | 1,000 | 300K | GPT-4.1 nano | $0.00007 | **$21** |
| PRO | 500 | 750K | Claude 3 Haiku | $0.000625 | **$469** |
| TEAM | 100 | 150K | Claude 3 Haiku | $0.000625 | **$94** |
| **Total** | 1,600 | 1.2M | Mixed | - | **$584** |

**Comparison with single model approach:**
- All Claude 3 Haiku: $750/month
- All GPT-4o-mini: $378/month
- **Tiered approach: $584/month** (22% savings vs all-Claude, better quality than all-GPT)

---

## Configuration Design

### Environment Variables

```bash
# Default models by tier
LLM_FREE_PRIMARY_PROVIDER=openai
LLM_FREE_PRIMARY_MODEL=gpt-4.1-nano
LLM_FREE_FALLBACK_PROVIDER=openai
LLM_FREE_FALLBACK_MODEL=gpt-4o-mini

LLM_PRO_PRIMARY_PROVIDER=anthropic
LLM_PRO_PRIMARY_MODEL=claude-3-haiku-20240307
LLM_PRO_FALLBACK_PROVIDER=openai
LLM_PRO_FALLBACK_MODEL=gpt-4o-mini

LLM_TEAM_PRIMARY_PROVIDER=anthropic
LLM_TEAM_PRIMARY_MODEL=claude-3-haiku-20240307
LLM_TEAM_FALLBACK_PROVIDER=openai
LLM_TEAM_FALLBACK_MODEL=gpt-4o-mini

# Global settings
LLM_TIMEOUT_MS=10000
LLM_MAX_RETRIES=2

# API Keys
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

### Easy Model Switching

```bash
# Upgrade Free tier to GPT-4o-mini after positive feedback
LLM_FREE_PRIMARY_MODEL=gpt-4o-mini

# Switch Pro tier to GPT-4.1 mini for cost savings
LLM_PRO_PRIMARY_PROVIDER=openai
LLM_PRO_PRIMARY_MODEL=gpt-4.1-mini

# A/B test: 50% users get new model (future feature)
LLM_AB_TEST_ENABLED=true
LLM_AB_TEST_MODEL=gpt-4.1-mini
LLM_AB_TEST_PERCENTAGE=50
```

---

## Implementation Tasks (12 Tasks)

### Task 1: Install Required SDKs

**Files:**
- Modify: `apps/backend/package.json`

**Step 1: Install SDKs**

Run:
```bash
cd apps/backend && pnpm add @anthropic-ai/sdk openai
```
Expected: Both packages added to dependencies

**Step 2: Verify installation**

Run:
```bash
cd apps/backend && pnpm list @anthropic-ai/sdk openai
```
Expected: Shows both SDK versions

**Step 3: Commit**

```bash
git add apps/backend/package.json apps/backend/pnpm-lock.yaml
git commit -m "feat: add Anthropic and OpenAI SDKs"
```

---

### Task 2: Create Tiered LLM Configuration

**Files:**
- Create: `apps/backend/src/config/llm.ts`
- Modify: `.env.example`

**Step 1: Create LLM config**

Create `apps/backend/src/config/llm.ts`:
```typescript
export type LLMProvider = 'anthropic' | 'openai' | 'google' | 'deepseek';
export type UserTier = 'free' | 'pro' | 'team' | 'enterprise';

export interface ModelConfig {
  provider: LLMProvider;
  model: string;
}

export interface TierConfig {
  primary: ModelConfig;
  fallback: ModelConfig;
}

export interface LLMConfig {
  tiers: Record<UserTier, TierConfig>;
  timeout: number;
  maxRetries: number;
}

function getEnv(key: string, defaultValue: string): string {
  return process.env[key] || defaultValue;
}

function getModelConfig(tierPrefix: string, type: 'PRIMARY' | 'FALLBACK', defaults: ModelConfig): ModelConfig {
  return {
    provider: getEnv(`LLM_${tierPrefix}_${type}_PROVIDER`, defaults.provider) as LLMProvider,
    model: getEnv(`LLM_${tierPrefix}_${type}_MODEL`, defaults.model),
  };
}

// Default configurations
const FREE_PRIMARY: ModelConfig = { provider: 'openai', model: 'gpt-4.1-nano' };
const FREE_FALLBACK: ModelConfig = { provider: 'openai', model: 'gpt-4o-mini' };
const PRO_PRIMARY: ModelConfig = { provider: 'anthropic', model: 'claude-3-haiku-20240307' };
const PRO_FALLBACK: ModelConfig = { provider: 'openai', model: 'gpt-4o-mini' };

export const llmConfig: LLMConfig = {
  tiers: {
    free: {
      primary: getModelConfig('FREE', 'PRIMARY', FREE_PRIMARY),
      fallback: getModelConfig('FREE', 'FALLBACK', FREE_FALLBACK),
    },
    pro: {
      primary: getModelConfig('PRO', 'PRIMARY', PRO_PRIMARY),
      fallback: getModelConfig('PRO', 'FALLBACK', PRO_FALLBACK),
    },
    team: {
      primary: getModelConfig('TEAM', 'PRIMARY', PRO_PRIMARY),
      fallback: getModelConfig('TEAM', 'FALLBACK', PRO_FALLBACK),
    },
    enterprise: {
      primary: getModelConfig('ENTERPRISE', 'PRIMARY', PRO_PRIMARY),
      fallback: getModelConfig('ENTERPRISE', 'FALLBACK', PRO_FALLBACK),
    },
  },
  timeout: parseInt(getEnv('LLM_TIMEOUT_MS', '10000'), 10),
  maxRetries: parseInt(getEnv('LLM_MAX_RETRIES', '2'), 10),
};

export function getTierConfig(tier: UserTier): TierConfig {
  return llmConfig.tiers[tier] || llmConfig.tiers.free;
}

// Validate required API keys
export function validateLLMConfig(): void {
  const requiredKeys: Partial<Record<LLMProvider, string>> = {
    anthropic: 'ANTHROPIC_API_KEY',
    openai: 'OPENAI_API_KEY',
  };

  // Collect all providers used across tiers
  const usedProviders = new Set<LLMProvider>();
  Object.values(llmConfig.tiers).forEach((tier) => {
    usedProviders.add(tier.primary.provider);
    usedProviders.add(tier.fallback.provider);
  });

  for (const provider of usedProviders) {
    const keyName = requiredKeys[provider];
    if (keyName && !process.env[keyName]) {
      throw new Error(`Missing required environment variable: ${keyName} for provider: ${provider}`);
    }
  }
}
```

**Step 2: Update .env.example**

Add to `.env.example`:
```bash
# ===========================================
# LLM Configuration (Tiered Model Strategy)
# ===========================================

# API Keys (Required)
OPENAI_API_KEY=your_openai_api_key
ANTHROPIC_API_KEY=your_anthropic_api_key

# Free Tier: Cost-optimized (GPT-4.1 nano)
# LLM_FREE_PRIMARY_PROVIDER=openai
# LLM_FREE_PRIMARY_MODEL=gpt-4.1-nano
# LLM_FREE_FALLBACK_PROVIDER=openai
# LLM_FREE_FALLBACK_MODEL=gpt-4o-mini

# Pro Tier: Quality-optimized (Claude 3 Haiku)
# LLM_PRO_PRIMARY_PROVIDER=anthropic
# LLM_PRO_PRIMARY_MODEL=claude-3-haiku-20240307
# LLM_PRO_FALLBACK_PROVIDER=openai
# LLM_PRO_FALLBACK_MODEL=gpt-4o-mini

# Team Tier: Premium quality
# LLM_TEAM_PRIMARY_PROVIDER=anthropic
# LLM_TEAM_PRIMARY_MODEL=claude-3-haiku-20240307
# LLM_TEAM_FALLBACK_PROVIDER=openai
# LLM_TEAM_FALLBACK_MODEL=gpt-4o-mini

# Global Settings
# LLM_TIMEOUT_MS=10000
# LLM_MAX_RETRIES=2
```

**Step 3: Commit**

```bash
git add apps/backend/src/config/llm.ts .env.example
git commit -m "feat: add tiered LLM configuration with per-tier model selection"
```

---

### Task 3: Create LLM Types with Cost Tracking

**Files:**
- Create: `apps/backend/src/services/llm/types.ts`

**Step 1: Create types file**

Create `apps/backend/src/services/llm/types.ts`:
```typescript
import { LLMProvider, UserTier } from '../../config/llm';

export type Tone =
  | 'formal'
  | 'casual'
  | 'professional'
  | 'persuasive'
  | 'friendly'
  | 'enthusiastic'
  | 'empathetic'
  | 'direct';

export interface ConversionRequest {
  text: string;
  tone: Tone;
  userTier: UserTier;
  userId: string;
}

export interface ConversionResult {
  text: string;
  provider: LLMProvider;
  model: string;
  inputTokens: number;
  outputTokens: number;
  latencyMs: number;
  cost: number;
  userTier: UserTier;
  usedFallback: boolean;
}

export interface LLMProviderInterface {
  convert(text: string, tone: Tone, model: string): Promise<Omit<ConversionResult, 'userTier' | 'usedFallback'>>;
}

// Cost rates per million tokens (updated 2026)
export const COST_RATES: Record<string, { input: number; output: number }> = {
  // OpenAI models
  'gpt-4.1-nano': { input: 0.02, output: 0.15 },
  'gpt-4o-mini': { input: 0.15, output: 0.60 },
  'gpt-4.1-mini': { input: 0.40, output: 1.60 },
  'gpt-4.1': { input: 2.00, output: 8.00 },
  'gpt-4o': { input: 2.50, output: 10.00 },

  // Anthropic models
  'claude-3-haiku-20240307': { input: 0.25, output: 1.25 },
  'claude-3-5-haiku-20241022': { input: 1.00, output: 5.00 },
  'claude-3-5-sonnet-20241022': { input: 3.00, output: 15.00 },

  // Google models (future)
  'gemini-1.5-flash': { input: 0.075, output: 0.30 },
  'gemini-2.0-flash': { input: 0.10, output: 0.40 },

  // DeepSeek models (future)
  'deepseek-chat': { input: 0.07, output: 0.27 }, // cached rate
};

export function calculateTokenCost(
  model: string,
  inputTokens: number,
  outputTokens: number
): number {
  const rates = COST_RATES[model] || COST_RATES['gpt-4o-mini'];
  return (inputTokens * rates.input + outputTokens * rates.output) / 1_000_000;
}

// For analytics and monitoring
export interface ConversionAnalytics {
  userId: string;
  userTier: UserTier;
  provider: LLMProvider;
  model: string;
  tone: Tone;
  inputTokens: number;
  outputTokens: number;
  cost: number;
  latencyMs: number;
  usedFallback: boolean;
  timestamp: Date;
}
```

**Step 2: Commit**

```bash
git add apps/backend/src/services/llm/types.ts
git commit -m "feat: add LLM types with tiered cost tracking"
```

---

### Task 4: Create Tone Prompts

**Files:**
- Create: `apps/backend/src/services/llm/prompts.ts`

**Step 1: Create prompts file**

Create `apps/backend/src/services/llm/prompts.ts`:
```typescript
import { Tone } from './types';

export const TONE_SYSTEM_PROMPTS: Record<Tone, string> = {
  formal:
    'You are a writing assistant. Rewrite the user\'s text in a formal, professional tone. Use proper grammar, avoid contractions, and maintain a respectful, business-appropriate style. Preserve the original meaning. Output only the rewritten text, nothing else.',

  casual:
    'You are a writing assistant. Rewrite the user\'s text in a casual, relaxed tone. Use conversational language, contractions are fine, and keep it friendly and approachable. Preserve the original meaning. Output only the rewritten text, nothing else.',

  professional:
    'You are a writing assistant. Rewrite the user\'s text in a professional tone suitable for business communication. Be clear, concise, and confident while maintaining warmth. Preserve the original meaning. Output only the rewritten text, nothing else.',

  persuasive:
    'You are a writing assistant. Rewrite the user\'s text in a persuasive tone. Use compelling language, emphasize benefits, and create a sense of importance. Preserve the original meaning. Output only the rewritten text, nothing else.',

  friendly:
    'You are a writing assistant. Rewrite the user\'s text in a friendly, warm tone. Be personable, use positive language, and make the reader feel valued. Preserve the original meaning. Output only the rewritten text, nothing else.',

  enthusiastic:
    'You are a writing assistant. Rewrite the user\'s text with enthusiasm and energy. Use exciting language and convey genuine excitement. Preserve the original meaning. Output only the rewritten text, nothing else.',

  empathetic:
    'You are a writing assistant. Rewrite the user\'s text with empathy and understanding. Acknowledge feelings, show compassion, and be supportive. Preserve the original meaning. Output only the rewritten text, nothing else.',

  direct:
    'You are a writing assistant. Rewrite the user\'s text in a direct, straightforward tone. Be clear and concise, get to the point quickly, and avoid unnecessary filler. Preserve the original meaning. Output only the rewritten text, nothing else.',
};

export const TONES: Tone[] = [
  'formal',
  'casual',
  'professional',
  'persuasive',
  'friendly',
  'enthusiastic',
  'empathetic',
  'direct',
];

export function isValidTone(tone: string): tone is Tone {
  return TONES.includes(tone as Tone);
}
```

**Step 2: Commit**

```bash
git add apps/backend/src/services/llm/prompts.ts
git commit -m "feat: add tone conversion prompts"
```

---

### Task 5: Implement OpenAI Provider

**Files:**
- Create: `apps/backend/src/services/llm/providers/openai.ts`

**Step 1: Create OpenAI provider**

Create `apps/backend/src/services/llm/providers/openai.ts`:
```typescript
import OpenAI from 'openai';
import { LLMProviderInterface, ConversionResult, Tone, calculateTokenCost } from '../types';
import { TONE_SYSTEM_PROMPTS } from '../prompts';

let client: OpenAI | null = null;

function getClient(): OpenAI {
  if (!client) {
    client = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });
  }
  return client;
}

export const openaiProvider: LLMProviderInterface = {
  async convert(
    text: string,
    tone: Tone,
    model: string
  ): Promise<Omit<ConversionResult, 'userTier' | 'usedFallback'>> {
    const startTime = Date.now();
    const openai = getClient();

    const response = await openai.chat.completions.create({
      model,
      max_tokens: 1024,
      messages: [
        {
          role: 'system',
          content: TONE_SYSTEM_PROMPTS[tone],
        },
        {
          role: 'user',
          content: text,
        },
      ],
    });

    const convertedText = response.choices[0]?.message?.content || '';
    const inputTokens = response.usage?.prompt_tokens || 0;
    const outputTokens = response.usage?.completion_tokens || 0;

    return {
      text: convertedText,
      provider: 'openai',
      model,
      inputTokens,
      outputTokens,
      latencyMs: Date.now() - startTime,
      cost: calculateTokenCost(model, inputTokens, outputTokens),
    };
  },
};
```

**Step 2: Commit**

```bash
git add apps/backend/src/services/llm/providers/openai.ts
git commit -m "feat: implement OpenAI provider with multi-model support"
```

---

### Task 6: Implement Anthropic Provider

**Files:**
- Create: `apps/backend/src/services/llm/providers/anthropic.ts`

**Step 1: Create Anthropic provider**

Create `apps/backend/src/services/llm/providers/anthropic.ts`:
```typescript
import Anthropic from '@anthropic-ai/sdk';
import { LLMProviderInterface, ConversionResult, Tone, calculateTokenCost } from '../types';
import { TONE_SYSTEM_PROMPTS } from '../prompts';

let client: Anthropic | null = null;

function getClient(): Anthropic {
  if (!client) {
    client = new Anthropic({
      apiKey: process.env.ANTHROPIC_API_KEY,
    });
  }
  return client;
}

export const anthropicProvider: LLMProviderInterface = {
  async convert(
    text: string,
    tone: Tone,
    model: string
  ): Promise<Omit<ConversionResult, 'userTier' | 'usedFallback'>> {
    const startTime = Date.now();
    const anthropic = getClient();

    const response = await anthropic.messages.create({
      model,
      max_tokens: 1024,
      system: TONE_SYSTEM_PROMPTS[tone],
      messages: [
        {
          role: 'user',
          content: text,
        },
      ],
    });

    const textContent = response.content.find((block) => block.type === 'text');
    const convertedText = textContent?.type === 'text' ? textContent.text : '';

    const inputTokens = response.usage.input_tokens;
    const outputTokens = response.usage.output_tokens;

    return {
      text: convertedText,
      provider: 'anthropic',
      model,
      inputTokens,
      outputTokens,
      latencyMs: Date.now() - startTime,
      cost: calculateTokenCost(model, inputTokens, outputTokens),
    };
  },
};
```

**Step 2: Commit**

```bash
git add apps/backend/src/services/llm/providers/anthropic.ts
git commit -m "feat: implement Anthropic/Claude provider"
```

---

### Task 7: Create Provider Registry

**Files:**
- Create: `apps/backend/src/services/llm/providers/index.ts`

**Step 1: Create provider registry**

Create `apps/backend/src/services/llm/providers/index.ts`:
```typescript
import { LLMProvider } from '../../../config/llm';
import { LLMProviderInterface } from '../types';
import { anthropicProvider } from './anthropic';
import { openaiProvider } from './openai';

const providers: Record<string, LLMProviderInterface> = {
  anthropic: anthropicProvider,
  openai: openaiProvider,
  // google: googleProvider,    // Future: Gemini
  // deepseek: deepseekProvider, // Future: DeepSeek
};

export function getProvider(provider: LLMProvider): LLMProviderInterface {
  const impl = providers[provider];
  if (!impl) {
    throw new Error(
      `Provider not implemented: ${provider}. Available: ${Object.keys(providers).join(', ')}`
    );
  }
  return impl;
}

export function isProviderAvailable(provider: LLMProvider): boolean {
  return provider in providers;
}

export function listAvailableProviders(): LLMProvider[] {
  return Object.keys(providers) as LLMProvider[];
}
```

**Step 2: Commit**

```bash
git add apps/backend/src/services/llm/providers/index.ts
git commit -m "feat: add provider registry"
```

---

### Task 8: Create Tiered Conversion Service

**Files:**
- Create: `apps/backend/src/services/llm/index.ts`

**Step 1: Create unified service**

Create `apps/backend/src/services/llm/index.ts`:
```typescript
import { llmConfig, getTierConfig, validateLLMConfig, UserTier } from '../../config/llm';
import { ConversionRequest, ConversionResult, Tone, ConversionAnalytics } from './types';
import { getProvider, isProviderAvailable } from './providers';
import { isValidTone } from './prompts';

// Re-export types and utilities
export { Tone, ConversionResult, ConversionRequest, ConversionAnalytics } from './types';
export { isValidTone, TONES } from './prompts';
export { UserTier } from '../../config/llm';

interface ConversionOptions {
  skipFallback?: boolean;
}

export async function convertTone(
  request: ConversionRequest,
  options: ConversionOptions = {}
): Promise<ConversionResult> {
  validateLLMConfig();

  if (!isValidTone(request.tone)) {
    throw new Error(`Invalid tone: ${request.tone}`);
  }

  const tierConfig = getTierConfig(request.userTier);
  const { primary, fallback } = tierConfig;
  const { timeout } = llmConfig;

  // Try primary provider
  if (isProviderAvailable(primary.provider)) {
    try {
      const provider = getProvider(primary.provider);
      const result = await Promise.race([
        provider.convert(request.text, request.tone, primary.model),
        new Promise<never>((_, reject) =>
          setTimeout(() => reject(new Error(`Primary provider timeout (${timeout}ms)`)), timeout)
        ),
      ]);

      const fullResult: ConversionResult = {
        ...result,
        userTier: request.userTier,
        usedFallback: false,
      };

      logConversion(request, fullResult);
      return fullResult;
    } catch (error) {
      console.warn(
        `[LLM] Primary provider (${primary.provider}/${primary.model}) failed for ${request.userTier} tier:`,
        error
      );

      if (options.skipFallback) {
        throw error;
      }
    }
  }

  // Try fallback provider
  if (isProviderAvailable(fallback.provider)) {
    try {
      const provider = getProvider(fallback.provider);
      const result = await provider.convert(request.text, request.tone, fallback.model);

      const fullResult: ConversionResult = {
        ...result,
        userTier: request.userTier,
        usedFallback: true,
      };

      logConversion(request, fullResult);
      console.info(`[LLM] Fallback to ${fallback.provider}/${fallback.model} succeeded`);
      return fullResult;
    } catch (fallbackError) {
      console.error(
        `[LLM] Fallback provider (${fallback.provider}/${fallback.model}) also failed:`,
        fallbackError
      );
      throw new Error('All LLM providers failed');
    }
  }

  throw new Error('No LLM providers available');
}

function logConversion(request: ConversionRequest, result: ConversionResult): void {
  console.log(
    `[LLM] Conversion: tier=${result.userTier}, provider=${result.provider}, model=${result.model}, ` +
    `tokens=${result.inputTokens}+${result.outputTokens}, cost=$${result.cost.toFixed(6)}, ` +
    `latency=${result.latencyMs}ms, fallback=${result.usedFallback}`
  );
}

// Health check for monitoring
export async function healthCheck(): Promise<{
  tiers: Record<UserTier, {
    primary: { available: boolean; provider: string; model: string };
    fallback: { available: boolean; provider: string; model: string };
  }>;
}> {
  const tiers = {} as Record<UserTier, any>;

  for (const [tier, config] of Object.entries(llmConfig.tiers)) {
    tiers[tier as UserTier] = {
      primary: {
        available: isProviderAvailable(config.primary.provider),
        provider: config.primary.provider,
        model: config.primary.model,
      },
      fallback: {
        available: isProviderAvailable(config.fallback.provider),
        provider: config.fallback.provider,
        model: config.fallback.model,
      },
    };
  }

  return { tiers };
}

// Get model info for a tier (useful for UI)
export function getModelInfoForTier(tier: UserTier): {
  provider: string;
  model: string;
  isPremium: boolean;
} {
  const config = getTierConfig(tier);
  return {
    provider: config.primary.provider,
    model: config.primary.model,
    isPremium: tier !== 'free',
  };
}
```

**Step 2: Commit**

```bash
git add apps/backend/src/services/llm/index.ts
git commit -m "feat: create tiered LLM conversion service with fallback"
```

---

### Task 9: Create Feedback Service

**Files:**
- Create: `apps/backend/src/services/feedback.ts`

**Step 1: Create feedback service**

Create `apps/backend/src/services/feedback.ts`:
```typescript
import { getFirestore, FieldValue } from 'firebase-admin/firestore';
import { UserTier } from '../config/llm';
import { Tone } from './llm/types';

export interface ConversionFeedback {
  id?: string;
  userId: string;
  userTier: UserTier;
  conversionId: string;
  rating: 1 | 2 | 3 | 4 | 5;
  model: string;
  provider: string;
  tone: Tone;
  comment?: string;
  createdAt: Date;
}

export interface FeedbackStats {
  model: string;
  averageRating: number;
  totalFeedback: number;
  ratingDistribution: Record<number, number>;
}

const COLLECTION = 'conversion_feedback';

export async function submitFeedback(
  feedback: Omit<ConversionFeedback, 'id' | 'createdAt'>
): Promise<string> {
  const db = getFirestore();
  const docRef = await db.collection(COLLECTION).add({
    ...feedback,
    createdAt: FieldValue.serverTimestamp(),
  });

  console.log(`[Feedback] Submitted: user=${feedback.userId}, model=${feedback.model}, rating=${feedback.rating}`);
  return docRef.id;
}

export async function getFeedbackStats(model?: string): Promise<FeedbackStats[]> {
  const db = getFirestore();
  let query = db.collection(COLLECTION);

  if (model) {
    query = query.where('model', '==', model) as any;
  }

  const snapshot = await query.get();
  const feedbackByModel = new Map<string, ConversionFeedback[]>();

  snapshot.forEach((doc) => {
    const data = doc.data() as ConversionFeedback;
    const existing = feedbackByModel.get(data.model) || [];
    existing.push(data);
    feedbackByModel.set(data.model, existing);
  });

  const stats: FeedbackStats[] = [];

  feedbackByModel.forEach((feedbacks, modelName) => {
    const ratings = feedbacks.map((f) => f.rating);
    const distribution: Record<number, number> = { 1: 0, 2: 0, 3: 0, 4: 0, 5: 0 };
    ratings.forEach((r) => distribution[r]++);

    stats.push({
      model: modelName,
      averageRating: ratings.reduce((a, b) => a + b, 0) / ratings.length,
      totalFeedback: ratings.length,
      ratingDistribution: distribution,
    });
  });

  return stats.sort((a, b) => b.averageRating - a.averageRating);
}

export async function getModelComparison(): Promise<{
  models: FeedbackStats[];
  recommendation: string;
}> {
  const stats = await getFeedbackStats();

  let recommendation = 'Not enough feedback data';
  if (stats.length >= 2) {
    const best = stats[0];
    const secondBest = stats[1];
    if (best.totalFeedback >= 10) {
      recommendation = `${best.model} (${best.averageRating.toFixed(2)} avg) outperforms ${secondBest.model} (${secondBest.averageRating.toFixed(2)} avg)`;
    }
  }

  return { models: stats, recommendation };
}
```

**Step 2: Commit**

```bash
git add apps/backend/src/services/feedback.ts
git commit -m "feat: add feedback service for model quality tracking"
```

---

### Task 10: Update Convert Route with Tier Support

**Files:**
- Modify: `apps/backend/src/routes/convert.ts`

**Step 1: Update route**

Update `apps/backend/src/routes/convert.ts`:
```typescript
import { Router, Request, Response } from 'express';
import { convertTone, isValidTone, TONES, Tone, getModelInfoForTier } from '../services/llm';
import { authMiddleware } from '../middleware/auth';
import { checkAndIncrementQuota } from '../services/quota';

const router = Router();

router.post('/', authMiddleware, async (req: Request, res: Response) => {
  try {
    const { text, tone } = req.body;
    const userId = req.user!.uid;
    const userTier = req.user!.tier || 'free';

    // Validate input
    if (!text || typeof text !== 'string' || text.trim().length === 0) {
      return res.status(400).json({ error: 'Text is required' });
    }

    if (text.length > 5000) {
      return res.status(400).json({ error: 'Text must be less than 5000 characters' });
    }

    if (!tone || !isValidTone(tone)) {
      return res.status(400).json({
        error: `Invalid tone. Available tones: ${TONES.join(', ')}`,
      });
    }

    // Check quota
    const quotaResult = await checkAndIncrementQuota(userId);
    if (!quotaResult.allowed) {
      return res.status(429).json({
        error: 'Daily quota exceeded',
        remaining: 0,
        tier: quotaResult.tier,
        upgradeUrl: '/pricing',
      });
    }

    // Convert tone with tier-appropriate model
    const result = await convertTone({
      text: text.trim(),
      tone: tone as Tone,
      userTier,
      userId,
    });

    return res.json({
      convertedText: result.text,
      metadata: {
        provider: result.provider,
        model: result.model,
        tier: result.userTier,
        tokens: {
          input: result.inputTokens,
          output: result.outputTokens,
        },
        latencyMs: result.latencyMs,
      },
      quota: {
        remaining: quotaResult.remaining - 1,
        tier: quotaResult.tier,
      },
    });
  } catch (error) {
    console.error('[Convert] Error:', error);
    return res.status(500).json({ error: 'Conversion failed. Please try again.' });
  }
});

// Get model info for current user's tier
router.get('/model-info', authMiddleware, async (req: Request, res: Response) => {
  const userTier = req.user!.tier || 'free';
  const modelInfo = getModelInfoForTier(userTier);

  return res.json({
    tier: userTier,
    ...modelInfo,
  });
});

export default router;
```

**Step 2: Commit**

```bash
git add apps/backend/src/routes/convert.ts
git commit -m "feat: update convert route with tiered model selection"
```

---

### Task 11: Create Feedback Route

**Files:**
- Create: `apps/backend/src/routes/feedback.ts`

**Step 1: Create feedback route**

Create `apps/backend/src/routes/feedback.ts`:
```typescript
import { Router, Request, Response } from 'express';
import { submitFeedback, getFeedbackStats, getModelComparison } from '../services/feedback';
import { authMiddleware } from '../middleware/auth';

const router = Router();

// Submit feedback for a conversion
router.post('/', authMiddleware, async (req: Request, res: Response) => {
  try {
    const { conversionId, rating, model, provider, tone, comment } = req.body;
    const userId = req.user!.uid;
    const userTier = req.user!.tier || 'free';

    if (!conversionId || !rating || !model || !provider || !tone) {
      return res.status(400).json({ error: 'Missing required fields' });
    }

    if (rating < 1 || rating > 5) {
      return res.status(400).json({ error: 'Rating must be between 1 and 5' });
    }

    const feedbackId = await submitFeedback({
      userId,
      userTier,
      conversionId,
      rating,
      model,
      provider,
      tone,
      comment,
    });

    return res.json({
      success: true,
      feedbackId,
      message: 'Thank you for your feedback!',
    });
  } catch (error) {
    console.error('[Feedback] Error:', error);
    return res.status(500).json({ error: 'Failed to submit feedback' });
  }
});

// Get feedback stats (admin only)
router.get('/stats', authMiddleware, async (req: Request, res: Response) => {
  try {
    // TODO: Add admin check
    const { model } = req.query;
    const stats = await getFeedbackStats(model as string | undefined);
    return res.json({ stats });
  } catch (error) {
    console.error('[Feedback] Error:', error);
    return res.status(500).json({ error: 'Failed to get stats' });
  }
});

// Get model comparison (admin only)
router.get('/compare', authMiddleware, async (req: Request, res: Response) => {
  try {
    // TODO: Add admin check
    const comparison = await getModelComparison();
    return res.json(comparison);
  } catch (error) {
    console.error('[Feedback] Error:', error);
    return res.status(500).json({ error: 'Failed to get comparison' });
  }
});

export default router;
```

**Step 2: Commit**

```bash
git add apps/backend/src/routes/feedback.ts
git commit -m "feat: add feedback routes for model quality tracking"
```

---

### Task 12: Update Documentation

**Files:**
- Modify: `docs/plans/2026-02-03-toneshift-mvp.md`
- Modify: `CLAUDE.md`

**Step 1: Update MVP plan**

Update the tech stack section in `2026-02-03-toneshift-mvp.md`:
```markdown
**Tech Stack:** React 18, TypeScript, Tailwind CSS, Vite, Node.js, Express, Firebase Auth, Firestore, **Tiered LLM System** (GPT-4.1 nano for Free, Claude 3 Haiku for Pro/Team), Stripe, Vercel, Browser Extensions (Chrome MV3, Firefox, Edge)
```

Update Cost Analysis section:
```markdown
## Cost Analysis (Tiered Model Strategy)

### Model Configuration by Tier
| Tier | Primary Model | Fallback | Cost/Conv |
|------|---------------|----------|-----------|
| FREE | GPT-4.1 nano | GPT-4o-mini | $0.00007 |
| PRO | Claude 3 Haiku | GPT-4o-mini | $0.000625 |
| TEAM | Claude 3 Haiku | GPT-4o-mini | $0.000625 |

### Monthly Cost Projection (1,000 Free + 500 Pro + 100 Team users)
| Tier | Conv/mo | Cost | Notes |
|------|---------|------|-------|
| FREE (1K users) | 300K | $21 | 10/day limit |
| PRO (500 users) | 750K | $469 | Unlimited |
| TEAM (100 users) | 150K | $94 | 5 users each |
| **Total** | 1.2M | **$584** | |

### Comparison
- Single model (Claude): $750/mo
- Single model (GPT-4o-mini): $378/mo
- **Tiered approach: $584/mo** (better quality/cost balance)
```

**Step 2: Update CLAUDE.md**

Change AI section:
```markdown
- **AI:** Tiered LLM System
  - Free: GPT-4.1 nano (cost-optimized)
  - Pro/Team: Claude 3 Haiku (quality-optimized)
  - Fallback: GPT-4o-mini (all tiers)
  - User feedback collection for quality monitoring
```

**Step 3: Commit**

```bash
git add docs/plans/2026-02-03-toneshift-mvp.md CLAUDE.md
git commit -m "docs: update to tiered LLM model strategy"
```

---

## Summary

### Architecture Benefits

1. **Cost Optimization by Tier**
   - Free users: $0.00007/conv (GPT-4.1 nano)
   - Pro/Team users: $0.000625/conv (Claude 3 Haiku)
   - 80%+ savings on free tier

2. **Quality Where It Matters**
   - Paying customers get best writing quality
   - Free users still functional for trial

3. **Feedback-Driven Improvement**
   - Collect user ratings per model
   - Data-driven model selection decisions
   - Easy A/B testing capability

4. **Flexible Configuration**
   - Change models via env vars (no code changes)
   - Per-tier customization
   - Easy to add new providers

### Configuration Examples

**Default (Tiered):**
```bash
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
# Uses defaults: nano for free, claude for pro/team
```

**Upgrade Free tier after positive feedback:**
```bash
LLM_FREE_PRIMARY_MODEL=gpt-4o-mini
```

**Switch Pro to GPT-4.1 mini for cost savings:**
```bash
LLM_PRO_PRIMARY_PROVIDER=openai
LLM_PRO_PRIMARY_MODEL=gpt-4.1-mini
```

### Monthly Cost Comparison

| Strategy | 1.2M Conv/mo | Quality |
|----------|--------------|---------|
| All Claude 3 Haiku | $750 | ⭐⭐⭐⭐⭐ all |
| All GPT-4o-mini | $378 | ⭐⭐⭐⭐ all |
| All GPT-4.1 nano | $84 | ⭐⭐⭐ all |
| **Tiered (this plan)** | **$584** | ⭐⭐⭐ free, ⭐⭐⭐⭐⭐ paid |

### Next Steps After Implementation

1. Launch with tiered configuration
2. Collect feedback from free users (1-2 weeks)
3. Analyze: Is GPT-4.1 nano quality acceptable?
4. If positive: Keep configuration
5. If negative: Upgrade free tier to GPT-4o-mini

---

## Sources

- [GPT-4.1 nano Model](https://platform.openai.com/docs/models/gpt-4.1-nano)
- [GPT-4.1 nano Pricing](https://gptbreeze.io/blog/gpt-41-nano-pricing-guide/)
- [GPT-4.1 mini vs GPT-4o-mini](https://blog.galaxy.ai/compare/gpt-4-1-mini-vs-gpt-4o-mini)
- [Claude 3 Haiku Pricing](https://www.anthropic.com/pricing)
- [OpenAI API Pricing](https://openai.com/api/pricing/)
- [LLM API Pricing Comparison](https://intuitionlabs.ai/articles/llm-api-pricing-comparison-2025)
