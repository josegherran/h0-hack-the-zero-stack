# ContractLens — OpenAI → Amazon Bedrock Migration Notes

> **Date:** June 2026
> **Version:** v1.0.0 → v1.1.0
> **Change:** Replaced OpenAI GPT-4o with Anthropic Claude 3.5 Sonnet via Amazon Bedrock

---

## Summary

All three documentation files (README.md, SPEC.md, ARCHITECTURE.md) have been updated to reflect the migration from OpenAI GPT-4o to **Anthropic Claude 3.5 Sonnet accessed via Amazon Bedrock**.

### Key Benefits

1. **No external API key for Lambda** — Lambda authenticates to Bedrock using its IAM execution role (`bedrock:InvokeModel`)
2. **Fully AWS-native** — All LLM calls stay within AWS; no cross-cloud traffic
3. **Claude's tool use** — Equivalent to OpenAI function calling; enforces typed JSON schema
4. **Vercel AI SDK support** — `@ai-sdk/amazon-bedrock` provider handles chat streaming seamlessly
5. **Cost efficiency** — Bedrock pricing is competitive; no separate vendor relationship

---

## Files Updated

### 1. README.md

**Changes:**
- Section 3.1: "AI Analysis" now references "Anthropic Claude 3.5 Sonnet via Amazon Bedrock"
- Section 5.1 (Stack table): Updated AI/LLM row with Bedrock details
- Section 5.2 (Architecture Diagram): Replaced `OpenAI` node with `Amazon Bedrock / Claude 3.5 Sonnet`
- Section 5.3 (Analysis Flow): Step 9 now calls "Anthropic Claude 3.5 Sonnet via Amazon Bedrock"
- Section 5.4 (Key Decisions): Added decision for "Amazon Bedrock for LLM" and updated "Vercel AI SDK for chat"
- Section 11 (Dev Plan): Day 2 task updated to reference Bedrock + Claude
- Section 12 (Risks): Updated "Bedrock / Claude cost during demo" (was "OpenAI cost")

### 2. SPEC.md

**Changes:**
- Version bumped: 1.0.0 → 1.1.0
- Section 3 (FR-02.2): Updated to reference Bedrock + Claude tool use
- Section 4 (FR-05.2): Updated to reference Vercel AI SDK + `@ai-sdk/amazon-bedrock`
- Section 4 (NFR-01.5): Updated chat latency target to reference Bedrock
- Section 4 (NFR-04.5, NFR-04.6): Updated IAM permissions and secrets handling
- Section 5 (Tech Stack): Updated AI/LLM and Real-time/Streaming rows
- Section 6 (Architecture Diagram): Replaced OpenAI with Bedrock
- Section 6 (Key Decisions): Added Bedrock decision, updated Vercel AI SDK decision
- Section 10 (Changelog): Added v1.1.0 entry documenting the migration
- Section 11 (Contributing): Updated env vars table; removed `OPENAI_API_KEY`, added `BEDROCK_MODEL_ID`
- Section 13 (Conclusion): Updated to emphasize AWS-native LLM pipeline

### 3. ARCHITECTURE.md

**Changes:**
- Version bumped: 1.0.0 → 1.1.0
- Section 3 (Patterns): Updated "Structured Output / Tool Use" to reference Claude tool use
- Section 4.1 (System Diagram): Replaced OpenAI node with Amazon Bedrock; updated all LLM call labels
- Section 4.3 (Analysis Pipeline Sequence): Updated to show `InvokeModel` call to Bedrock instead of ChatCompletion to OpenAI
- Section 5 (Layers): Updated Layer 3 component table to reference Bedrock client
- Section 6.2 (Lambda Worker): Detailed explanation of tool use, IAM role auth, and Bedrock integration
- Section 6.7 (Vercel AI SDK): Updated to explain `@ai-sdk/amazon-bedrock` provider and environment variable auth
- Section 7 (ADRs): 
  - ADR-002 context updated to reference Claude instead of GPT-4o
  - ADR-007 replaced with new ADR-008 for Bedrock LLM provider decision
- Section 8 (Tech Stack): Updated AI SDK and LLM rows
- Section 9 (Improvements): Updated "Fine-tuned model" entry to reference Bedrock Custom Models
- Section 10 (Compliance): Updated IAM permissions and secrets handling

---

## Environment Variables

### Removed
- `OPENAI_API_KEY` — No longer needed

### Added
- `BEDROCK_MODEL_ID` — Bedrock model ID (e.g., `anthropic.claude-3-5-sonnet-20241022-v2:0`)

### Unchanged (still required)
- `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
- `DATABASE_URL`
- `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- `S3_BUCKET_NAME`, `SQS_QUEUE_URL`, `SES_FROM_EMAIL`

### Note on Lambda Authentication
Lambda→Bedrock calls use the **Lambda IAM execution role** with `bedrock:InvokeModel` permission. No API key is needed in the Lambda environment.

Vercel API Routes (chat endpoint) require `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` because Vercel cannot use IAM roles directly.

---

## AWS Setup Requirements

### 1. Enable Bedrock in Your Region

1. Go to AWS Console → Amazon Bedrock
2. Click "Get started" or navigate to "Model access"
3. Request access to **Claude 3.5 Sonnet** (Anthropic)
4. Wait for approval (usually instant)

### 2. Update Lambda IAM Role

Add the following inline policy to the Lambda execution role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:*:ACCOUNT_ID:foundation-model/anthropic.claude-3-5-sonnet-*"
    }
  ]
}
```

Replace `ACCOUNT_ID` with your AWS account ID.

### 3. Verify Bedrock Model ARN

The model ARN format is:
```
arn:aws:bedrock:REGION:ACCOUNT_ID:foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0
```

Verify the exact model ID in the Bedrock console under "Model access".

---

## Code Changes Required

### Lambda Analysis Worker

**Before:**
```typescript
import { OpenAI } from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: prompt }],
  response_format: { type: "json_object" },
});
```

**After:**
```typescript
import { BedrockRuntimeClient, InvokeModelCommand } from "@aws-sdk/client-bedrock-runtime";

const bedrock = new BedrockRuntimeClient({ region: process.env.AWS_REGION });
const response = await bedrock.send(new InvokeModelCommand({
  modelId: process.env.BEDROCK_MODEL_ID,
  body: JSON.stringify({
    anthropic_version: "bedrock-2023-06-01",
    max_tokens: 4096,
    system: systemPrompt,
    messages: [{ role: "user", content: prompt }],
    tools: [{ name: "analysis", input_schema: analysisSchema }],
  }),
}));
```

### Vercel API Route (Chat)

**Before:**
```typescript
import { openai } from "@ai-sdk/openai";
import { streamText } from "ai";

const result = await streamText({
  model: openai("gpt-4o"),
  system: systemPrompt,
  messages: userMessages,
});
```

**After:**
```typescript
import { bedrock } from "@ai-sdk/amazon-bedrock";
import { streamText } from "ai";

const result = await streamText({
  model: bedrock("anthropic.claude-3-5-sonnet-20241022-v2:0"),
  system: systemPrompt,
  messages: userMessages,
});
```

---

## Testing Checklist

- [ ] Lambda can invoke Bedrock (check CloudWatch logs for `InvokeModel` calls)
- [ ] Analysis returns valid JSON with risk_score, findings, key_dates
- [ ] Chat endpoint streams responses correctly
- [ ] Tool use schema is enforced (invalid JSON is rejected)
- [ ] No `OPENAI_API_KEY` references in code or environment
- [ ] Bedrock model access is enabled in the target region
- [ ] Lambda IAM role has `bedrock:InvokeModel` permission

---

## Rollback Plan

If you need to revert to OpenAI:

1. Revert the three documentation files to v1.0.0
2. Restore `OPENAI_API_KEY` environment variable
3. Revert Lambda code to use OpenAI SDK
4. Revert Vercel API Route to use `@ai-sdk/openai`
5. Remove `bedrock:InvokeModel` from Lambda IAM role

All changes are backward-compatible; no database schema changes were made.

---

## References

- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [Anthropic Claude API](https://docs.anthropic.com/claude/reference/getting-started-with-the-api)
- [Vercel AI SDK — Amazon Bedrock Provider](https://sdk.vercel.ai/docs/reference/ai-sdk-amazon-bedrock)
- [AWS SDK for JavaScript — Bedrock Runtime](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/v3/latest/client/bedrock-runtime/index.html)

---

*ContractLens Migration Notes — v1.0.0 → v1.1.0 — June 2026*
