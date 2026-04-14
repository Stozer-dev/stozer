# Stozer

**Grounding validation for tool-calling AI agents and RAG pipelines.**

> Detect when an agent's response contradicts its tool outputs or retrieved context — deterministically, without LLM-as-judge overhead.

[![npm version](https://img.shields.io/npm/v/stozer-ai.svg)](https://www.npmjs.com/package/stozer-ai)

---

## The Problem

Most "hallucinations" in tool-calling agents and RAG pipelines are **grounding failures** — the agent gets accurate data from tools or retrieved context, and then ignores it, miscalculates, or fabricates from empty results. The source of truth is already in the trace.

## Why Not LLM-as-a-Judge?

| | Stozer | LLM-as-a-Judge |
|---|---|---|
| **Latency** | <50ms | 2–10s per call |
| **Cost** | $0 compute | $0.01–0.10 per eval |
| **Determinism** | Same input → same result | Varies across runs |
| **Explainability** | Exact failure code + evidence | "I think there might be an issue" |
| **Hallucination risk** | Zero (no LLM) | The judge can hallucinate too |
| **Scalability** | 10K+ evals/sec | Rate-limited by provider |

## The Solution

Stozer validates the agent's response against tool outputs and reports grounding failures with standardized codes — like OBD diagnostic codes for AI.

```
npm install stozer-ai
```

**Zero LLM calls.** Deterministic failure detection. Runs in <50ms.

---

## Getting Started

1. **Create a free account** at [app.stozer.dev](https://app.stozer.dev) — no credit card required
2. **Copy your API key** from Settings → API Keys (starts with `stozer_live_`)
3. **Set the env var** or save it with the CLI:

```bash
# Option A: environment variable
export STOZER_API_KEY=stozer_live_your_key

# Option B: save once with the CLI (persists across sessions)
npx stozer auth stozer_live_your_key
```

The free tier includes full access to grounding evaluation, failure detection, and the dashboard — enough to integrate and test with your agent.

---

## Quick Start — 3 Minutes

### 1. Evaluate a trace

```typescript
import { StozerClient, TraceBuilder } from 'stozer-ai';

const client = new StozerClient();  // uses STOZER_API_KEY env var

const trace = new TraceBuilder({ traceId: 'run-001' })
  .addUserInput('How many employees are on leave today?')
  .addToolCall('getLeaveRecords', { date: '2024-03-15' })
  .addToolOutput('getLeaveRecords', [
    { employeeId: 'E01', name: 'Sarah Johnson', status: 'on_leave' },
    { employeeId: 'E02', name: 'Michael Chen', status: 'on_leave' },
  ])
  .addFinalResponse('There are 3 employees on leave today.')  // ← Bug: says 3, data shows 2
  .build();

const result = await client.evaluate(trace);

console.log(result.report.groundingScore);       // 0.5
console.log(result.report.detectedFailures[0]);  // { type: 'grounding.data_ignored', severity: 'high' }
```

### 2. Monitor in production (proxy mode)

Works with **any language** — PHP, Python, Go, Java, Ruby, C#.

Change your AI base URL and add your Stozer key using the composite key format (`||`):
```bash
# OpenAI — add Stozer key before your AI key, separated by ||
OPENAI_BASE_URL=http://localhost:3001/proxy/openai/v1
OPENAI_API_KEY=stozer_live_your_key||sk-your-openai-key

# Anthropic
ANTHROPIC_BASE_URL=http://localhost:3001/proxy/anthropic
ANTHROPIC_API_KEY=stozer_live_your_key||sk-ant-your-key
```

Your app works exactly the same — zero code changes. Stozer automatically splits the key, identifies your account, and forwards only the AI key to the provider.

---

## Detection

Deterministic failure detectors across six categories: **grounding**, **reasoning**, **orchestration**, **safety**, **semantic**, and **instrumentation**.

Examples:
- Fabrication from empty tool results or missing RAG context
- Numeric inconsistencies with tool data
- Ignored or altered data in the response
- Entity misattribution
- Prompt leakage and sensitive data exposure
- Tool budget exhaustion and fabricated tool inputs

---

## Features

### Diagnostic Advisor

Every detected failure includes actionable diagnostics — root cause identification, evidence, and remediation guidance.

### Policy Engine

Configure per-failure actions on your Stozer dashboard — block, warn, or observe for each failure type. SDK wrappers automatically enforce your configured policy:

```typescript
import { wrapOpenAI, GroundingError } from 'stozer-ai';
import OpenAI from 'openai';

const openai = wrapOpenAI(new OpenAI(), {
  mode: 'block',
  threshold: 0.85,
  onEvaluate: (report, trace) => {
    console.log('Score:', report.groundingScore);
    console.log('Failures:', report.detectedFailures.length);
  },
});
```

Also available: `wrapAnthropic` for Anthropic SDK, `wrapGemini` for Google Gemini SDK.

```typescript
import { wrapAnthropic } from 'stozer-ai';
const client = wrapAnthropic(new Anthropic(), { mode: 'observe' });

import { wrapGemini } from 'stozer-ai';
const model = wrapGemini(genAI.getGenerativeModel({ model: 'gemini-2.0-flash' }), { mode: 'observe' });
```

### MCP Server (VS Code, Cursor, Windsurf, Zed, Claude Desktop)

Use Stozer directly from your IDE — no terminal needed.

**Setup (one command):**
```bash
npx stozer auth stozer_live_your_key
npx stozer mcp-setup
```

Auto-detects installed IDEs and configures MCP. Restart your IDE after running.

<details>
<summary>Alternative: Manual setup</summary>

1. In VS Code: `Ctrl+Shift+P` → **"MCP: Open User Configuration"**
2. Add this to `mcp.json`:

```json
{
  "servers": {
    "stozer": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "stozer-ai", "mcp"]
    }
  }
}
```

3. Restart VS Code
</details>

**Usage:** In Copilot Chat, say: *"Call stozer verify_response with this trace: {...}"*

8 tools available: `verify_response`, `quick_check`, `check_trace_quality`, `list_rules`, `get_failure_info`, `evaluate_with_policy`, `get_live_traces`, `get_trace_report`

### Express Middleware

```typescript
import express from 'express';
import { groundingMiddleware, FileStore } from 'stozer-ai';

const app = express();
app.post('/api/chat', groundingMiddleware({
  mode: 'warn',
  store: new FileStore('./traces/grounding.jsonl'),
  extractTrace: (req, res, body) => body.trace,
}));
```

---

## CLI

```bash
npx stozer auth stozer_live_your_key      # Save API key (once)
npx stozer mcp-setup                      # Auto-configure MCP for all IDEs
npx stozer evaluate trace.json            # Evaluate one trace
npx stozer traces                         # List recent evaluations
npx stozer trace <trace-id>               # Full report for a specific trace
npx stozer rules [category]               # List detection rules (grounding, reasoning, safety, ...)
npx stozer rule <failure_type>            # Details + remediation for a rule
```

> **Auto-discovery:** Set `STOZER_API_KEY` env var and all SDK wrappers automatically send traces — no explicit client setup needed.

---

## How It Works

1. **Analyze** the agent's response against tool output data
2. **Detect** grounding failures using deterministic rules
3. **Score** overall grounding quality
4. **Diagnose** root causes with actionable remediation

**No LLM calls.** Deterministic. Supports 11 human languages.

---

## Benchmarks

Tested on public hallucination datasets — no cherry-picking, full results.

| Benchmark | Samples | Precision | Recall | F1 | Runtime |
|---|---|---|---|---|---|
| **HaluEval QA** | 16,662 | 99.98% | 98.6% | **99.3%** | 27s |
| **FaithBench** (w/ NLI) | 750 | 59.9% | 96.8% | 74.0% | 1,018s |
| **Production traces** | — | 97.9% | 92.0% | **94.8%** | — |

**HaluEval QA** (Li et al., 2023): 16,662 question–answer pairs with known hallucinations. Stozer detected 98.6% of hallucinated answers with near-zero false positives (2 out of 16K samples).

**FaithBench** (Bai et al., 2024): 750 free-text summaries. Lower precision is expected — FaithBench tests paraphrased prose, not structured tool outputs. Stozer is optimized for tool-calling and RAG traces where the source of truth is structured data.

**Production traces**: ~200 manually verified agent traces from production deployments across HR, finance, and operations domains.

Read the full benchmark report at [stozer.dev/blog/benchmarks](https://stozer.dev/blog/benchmarks).

---

## Links

- **Website:** [stozer.dev](https://stozer.dev)
- **Dashboard:** [app.stozer.dev](https://app.stozer.dev)
- **npm:** [stozer-ai](https://www.npmjs.com/package/stozer-ai)
- **Benchmark report:** [stozer.dev/blog/benchmarks](https://stozer.dev/blog/benchmarks)

---

## License

Proprietary. The npm package (`stozer-ai`) is freely installable. The detection engine runs server-side.
