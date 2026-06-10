# Bedrock Token Bill Retrofit: Prompt Caching + Intelligent Prompt Routing to Stop Rebilling the Same Tokens

Retrofits your existing Amazon Bedrock app with cache checkpoints on the stable prompt prefix and a quality-anchored prompt router—so the 8,000-token system prompt you send 50,000 times a day stops billing at full price, and simple prompts stop landing on your most expensive model.

## The Problem

Bedrock bills every input token on every request. A support assistant sending a 6,000-token system prompt, 2,000 tokens of tool definitions, and ~300 variable user tokens per request (8,300 tokens total)—50,000 requests a day on Claude Sonnet 4.5 at $3.00 per million input tokens—pays 415M input tokens/day = $1,245/day, about **$37,350/month**, and roughly 96% of those tokens are byte-identical between requests. Add ~$9,000/month of output tokens and you are at $46,350/month. The default reaction is AWS Budgets alarms, which only tell you you overspent after the fact. Bedrock ships two billing *mechanisms* that fix the spend itself: **prompt caching** (cache reads bill at $0.30/M on Sonnet 4.5—a 90% discount versus standard input, with cache writes at $3.75/M, +25%, per the Amazon Bedrock pricing page) and **Intelligent Prompt Routing** (route each request to the cheapest model in a family that clears a measured quality bar—AWS's published benchmark is up to 30% savings). Most teams skip both because checkpoint placement rules are fiddly and the routing trade-offs are poorly understood. This prompt does the retrofit correctly, with verification.

## Who This Is For

Startups running chat, RAG, or agent workloads on Bedrock on-demand inference spending $500+/month on tokens, with prompts that repeat context (system prompts, tool schemas, policy/reference documents) across requests. Not for batch workloads—prompt caching is not supported by the batch inference API (use the batch 50% discount there instead).

## How to Use

1. Copy the System Prompt below into Kiro CLI, Claude Code, or Amazon Q Developer CLI in a session with AWS credentials for the target account—or save it as `.kiro/steering/bedrock-cost-cutter.md`.
2. Replace the bracketed placeholders: `[REGION]`, `[MODEL_ID]`, `[REQUESTS_PER_DAY]`, `[CODE_PATH]`.
3. Point the assistant at the file that builds your Bedrock `Converse`/`InvokeModel` requests and let it run the reality checks (model support, Region, SDK version) before it writes anything.
4. Review the generated diff and the before/after cost model, then run `python verify_cache.py`—it must show `cacheReadInputTokens > 0` on the second call before you deploy.
5. Create the router with the generated `create-router.sh`, shadow-test it on 200 real prompts, and import `dashboard.json` into CloudWatch.

**Prerequisites**
- **Required Access:** IAM principal with `bedrock:InvokeModel`, `bedrock:ListFoundationModels`, `bedrock:CreatePromptRouter`, `bedrock:GetPromptRouter`, `bedrock:ListPromptRouters`, `cloudwatch:PutMetricData`, `cloudwatch:PutDashboard`, `cloudwatch:PutMetricAlarm` (scope down from `AmazonBedrockFullAccess`); model access granted on the Bedrock console **Model access** page.
- **Recommended Background:** Basic familiarity with the Bedrock Converse API and your app's request-building code.
- **Tools Required:** AWS CLI v2, Python 3.11+ with boto3 >= 1.37, AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) for live verification of model caching minimums and router Region support.

**Key Parameters:** Cache TTL (`5m` default, `"ttl": "1h"` on Claude Sonnet 4.5 / Haiku 4.5 / Opus 4.5 for sessions idle >5 min), checkpoint minimum (4,096 tokens for Sonnet 4.5 / Haiku 4.5 / Opus 4.5; verify per-model via docs; max 4 checkpoints), `responseQualityDifference` (start 0.3), cache hit-rate alarm (<60% for 3 datapoints), premium-model routing share alarm (>50%), target bill reduction (40%).

**Troubleshooting:** If `cacheReadInputTokens` stays 0 while `cacheWriteInputTokens` keeps climbing, your "static" prefix isn't static—a timestamp, request ID, or non-deterministically ordered tool list is mutating the prefix and forcing a cache rewrite on every call. Byte-diff two consecutive request payloads; the mutating field is the bug.

## System Prompt

```
# Bedrock Inference Cost Optimization Request: Prompt Caching + Intelligent Prompt Routing

## Project Overview
Act as a senior AWS GenAI cost engineer. Retrofit my existing Amazon Bedrock application in [REGION] (primary model [MODEL_ID], roughly [REQUESTS_PER_DAY] requests/day, client code at [CODE_PATH]) with prompt caching and Intelligent Prompt Routing. Cut inference spend with billing mechanisms, not budget alarms. Output must be production-ready: verified against my real account, reversible, and measured.

## Verify Reality First (hard requirement — do this before writing any code)
1. Run `aws bedrock list-foundation-models --region [REGION] --by-inference-type ON_DEMAND` and confirm [MODEL_ID] exists and supports prompt caching. Confirm its checkpoint minimum with aws_read_documentation on the Bedrock prompt-caching guide before assuming a value — at the time of writing, Claude Sonnet 4.5, Claude Haiku 4.5, and Claude Opus 4.5 each require 4,096 tokens per checkpoint, with a maximum of 4 checkpoints per request, but minimums are per-model and change as models are added.
2. Run `aws bedrock list-prompt-routers --region [REGION]` to confirm Intelligent Prompt Routing availability and list default routers. A router pairs exactly two models from one family (Anthropic Claude, Meta Llama, or Amazon Nova).
3. Confirm `boto3 >= 1.37` in my environment; older clients reject `cachePoint` with ParamValidationError.
4. Read my actual request-building code and tag every content block STATIC (byte-identical across requests) or DYNAMIC. If you cannot read the code, ask me for one representative Converse request body and STOP until you have it.
If any check fails, report the blocker and the fix. Never generate code against assumptions.

## Detailed Requirements

### 1. Prompt Caching — checkpoint placement rules
- Restructure each request so the stable prefix comes first, in Bedrock's processing order: `tools` -> `system` -> reference documents at the head of `messages`. Checkpoints chain in that order, and the minimum token count is evaluated cumulatively across all three sections; editing an earlier section invalidates every cache after it, so the most stable content goes earliest.
- Place one `cachePoint {"type": "default"}` immediately after the last stable block. Move ALL volatile content—timestamps, request IDs, retrieved RAG chunks, conversation turns—after the checkpoint. For Claude models, rely on Bedrock's simplified cache management (automatic prefix matching looks back ~20 content blocks from a single checkpoint); use multiple checkpoints only for sections that change at different frequencies.
- If the cumulative prefix misses the model minimum (4,096 tokens on Sonnet 4.5 / Haiku 4.5 / Opus 4.5—verify per-model), Bedrock silently skips caching. Reach the minimum by moving real reference content (policy docs, schemas) into the prefix—never pad with junk tokens.
- Default 5-minute TTL refreshes on every hit, so steady traffic keeps the cache warm for free. Use `"ttl": "1h"` only on Sonnet 4.5 / Haiku 4.5 / Opus 4.5 for sessions idle >5 minutes, and place 1h checkpoints before 5m ones (longer TTLs must come first). Note in the runbook that 1-hour writes bill above 5-minute writes.

### 2. Intelligent Prompt Routing — state the trade-offs honestly
- Generate `create-router.sh` using `aws bedrock create-prompt-router` with the required `--prompt-router-name`, `--models` (each model's ARN), `--fallback-model`, and `--routing-criteria '{"responseQualityDifference": 0.3}'`. The fallback model is the quality anchor: the router switches to the cheaper model only when its predicted response quality is within the threshold. Start at 0.3; lower it if quality dips, raise it to push more traffic to the cheap model.
- Invoke the router by passing its `promptRouterArn` as the Converse `modelId`. Log `trace.promptRouter.invokedModelId` from every response.
- Write a LIMITS section in the runbook stating plainly: routing is optimized for English prompts only; it cannot learn from my application's quality data; it adds $1 per 1,000 requests on top of model tokens, so show break-even math against my volume; and cache entries are per-model, so routing across two models fragments cache hit rates—quantify that interaction for my traffic and recommend whether to cache, route, or both per call path.
- Require a shadow evaluation on 200 representative production prompts comparing router output vs. the anchor model before any production cutover.

### 3. Measurement
- Parse `usage.cacheReadInputTokens`, `usage.cacheWriteInputTokens`, and `usage.inputTokens` from every Converse response (total input = sum of all three) and publish a custom CloudWatch metric `BedrockCost/CacheHitRate` via `put-metric-data`.
- Build one CloudWatch dashboard (4 widgets: cache hit rate, cached vs. uncached input tokens, routing share per model, estimated $/1K requests) and 2 alarms: CacheHitRate < 60% for 3 consecutive 5-minute datapoints, and premium-model routing share > 50% for 1 hour.

### 4. Cost Target
- Model the before/after monthly bill from my real traffic using current Amazon Bedrock pricing-page rates (on Claude Sonnet 4.5: $3.00/M input, $15.00/M output, $3.75/M cache write, $0.30/M cache read). Target >= 40% total bill reduction. Reject any change that worsens p95 time-to-first-token; cache hits should improve it.

## Error Management
- `cacheReadInputTokens` stuck at 0: byte-diff two consecutive request payloads; find and pin the mutating field before touching anything else.
- ValidationException from create-prompt-router: model pair or Region unsupported—print my supported alternatives from list-prompt-routers.
- AccessDeniedException on InvokeModel: model access not granted on the Bedrock console Model access page, or missing bedrock:InvokeModel on the router/model ARN.
- ThrottlingException after rollout: cache read tokens do not count against token rate limits, so check whether the router's cheap model has lower quotas than the anchor.

## Deliverables Requested
1. `cost-baseline.md` — current spend math from my measured traffic
2. Patched client code as a unified diff (cachePoint placement + router invocation behind a feature flag)
3. `create-router.sh` plus the exact `aws bedrock delete-prompt-router` rollback command
4. `verify_cache.py` — calls Converse twice with the restructured prompt and asserts cacheReadInputTokens > 0 on call two, printing effective per-call input cost
5. `dashboard.json` + `put-metric-alarm` CLI for both alarms
6. `runbook.md` — trade-off table, break-even math, shadow-eval results template, rollback steps

Output the deliverables directly without preamble. Every dollar figure must trace to the Bedrock pricing page or my measured traffic—never invent a price. Done means: verify_cache.py passes, the router answers with a trace showing the selected model, and the modeled bill shows >= 40% reduction—full retrofit applied in under 30 minutes.
```

## What You Get

1. **`cost-baseline.md`** — your current per-day and per-month input/output token spend, computed from real traffic numbers
2. **A unified diff** against your Bedrock client code: restructured `tools` -> `system` -> `messages` prefix, `cachePoint` placement, router invocation behind a feature flag
3. **`create-router.sh`** — the exact `aws bedrock create-prompt-router` command with anchor model, cheap model, and `responseQualityDifference`, plus the one-line rollback
4. **`verify_cache.py`** — two-call proof that caching is live (`cacheReadInputTokens > 0`), with effective input price printed per call
5. **`dashboard.json` + two alarm commands** — cache hit rate, cached vs. uncached tokens, routing share, $/1K requests
6. **`runbook.md`** — honest trade-off table (English-only routing, per-model cache fragmentation, $1/1K routing fee break-even), shadow-eval template, rollback procedure

## Example Output

```
$ python verify_cache.py --region us-east-1 --model anthropic.claude-sonnet-4-5-20250929-v1:0
Call 1: inputTokens=312   cacheWriteInputTokens=8,142   cacheReadInputTokens=0
Call 2: inputTokens=298   cacheWriteInputTokens=0       cacheReadInputTokens=8,142
PASS — cache hit confirmed. Effective input cost call 2: $0.0033 vs $0.0253 uncached (-87%)
Router probe ("reset my password"): trace.promptRouter.invokedModelId = claude-haiku-4-5 (cheap tier)
Router probe (1,900-token refund-policy edge case): invokedModelId = claude-sonnet-4-5 (anchor)
Modeled bill: $46,350/mo -> $12,030/mo (-74%). Routing fee at 50,000 req/day: $1,500/mo (net positive).
```

## AWS Services Used

- **Amazon Bedrock** — prompt caching (Converse `cachePoint` / InvokeModel `cache_control`), Intelligent Prompt Routing (`create-prompt-router`, `list-prompt-routers`), Converse API
- **Amazon CloudWatch** — custom `BedrockCost/CacheHitRate` metric, 4-widget dashboard, 2 alarms
- **AWS IAM** — least-privilege policy scoped to specific model and router ARNs
- **AWS CLI** — reality checks and router lifecycle

## Well-Architected Alignment

- **Cost Optimization:** attacks the two largest GenAI cost drivers—repeated input tokens (90% cache-read discount, official Bedrock pricing basis) and over-provisioned model selection (router with explicit break-even math including the $1/1K routing fee)
- **Performance Efficiency:** cache hits skip prefix recomputation, cutting time-to-first-token on long prompts; cache reads also don't count against token rate limits, raising effective throughput
- **Operational Excellence:** every change ships with a verification script, dashboard, alarms, and a rollback command; savings are measured, not assumed
- **Reliability:** the router's fallback model is an explicit quality anchor; caching composes with cross-region inference (expect marginally more cache writes at peak demand—documented behavior, covered in the runbook)

## Cost Notes

Basis: Amazon Bedrock pricing page, us-east-1 on-demand. Claude Sonnet 4.5: $3.00/M input, $15.00/M output, cache write $3.75/M (+25%), cache read $0.30/M (90% discount). Claude Haiku 4.5: $1.00/M input, $5.00/M output, cache read $0.10/M. Worked example at 50,000 req/day with an 8,000-token stable prefix + 300 variable tokens: input spend falls from $1,245/day ($37,350/mo) to ~$170/day ($120 cache reads + $45 variable + ~$5 warm-cache writes), about $5,100/mo—an 86% input reduction. Routing 60% of traffic to Haiku 4.5 then cuts output spend ~$3,600/mo against a $1,500/mo routing fee ($1 per 1,000 requests), netting ~$2,100/mo more. Total: $46,350/mo -> ~$12,000/mo. Honest floor: routing rarely pays on Nova Micro-class traffic, where a whole request costs ~$0.0001 against the $0.001 fee—the prompt computes this break-even for your numbers instead of assuming savings.

## Troubleshooting

- **`cacheReadInputTokens` always 0** — Cause: prefix below the model minimum (4,096 tokens on Sonnet 4.5/Haiku 4.5; Bedrock silently skips the checkpoint) or a mutating field (timestamp, UUID, unsorted tool list) in the prefix. Fix: byte-diff two consecutive payloads; move real reference content into the prefix to clear the minimum.
- **`ParamValidationError: Unknown parameter cachePoint`** — Cause: boto3/botocore predates prompt caching GA. Fix: `pip install --upgrade boto3` (>= 1.37) and redeploy.
- **`ValidationException` on `create-prompt-router`** — Cause: model pair or Region unsupported, not exactly two models from one family, or a missing required argument (`--prompt-router-name`, `--fallback-model`, `--models`, `--routing-criteria`). Fix: run `aws bedrock list-prompt-routers`, pick a supported pair (e.g., Claude family in us-east-1/us-west-2), keep anchor and alternate in the same family, and supply all four required arguments.
- **Router sends nearly everything to the expensive model** — Cause: `responseQualityDifference` too strict, or non-English traffic (routing is optimized for English only). Fix: raise the threshold from 0.1 toward 0.3–0.5 and re-run the 200-prompt shadow eval; pin non-English paths directly to one model.
- **Cache hit rate dropped after enabling routing** — Cause: cache entries are per-model; traffic split across two models fragments the prefix cache. Fix: cache-only on the high-volume anchor path, or accept the measured hit-rate cost if routing's savings dominate—the dashboard shows both.
