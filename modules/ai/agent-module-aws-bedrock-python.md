# Module: AWS Bedrock Integration Standard (Python)

## 1. Module Metadata
- Module ID: `agent.module.aws-bedrock-python`
- Version: `1.0.0`
- Maturity: `production`
- Scope: Secure, scalable Bedrock integration for LLM inference and embeddings in Python services.
- Primary outcomes:
  - Deterministic Bedrock client construction.
  - Safe credential and model access management.
  - Production-grade inference, error handling, and cost controls.

## 2. Mission and Applicability
Use this module to integrate AWS Bedrock into backend services, AI pipelines, and RAG workloads.

Apply when:
- Python runtime is used.
- Bedrock provides text generation and/or embeddings.
- Credentials must come from environment or IAM runtime chain.

Do not apply directly when:
- Runtime is non-Python.
- Inference platform is not AWS Bedrock.

## 3. Operating Principles
- Never hard-code credentials.
- Keep model IDs configurable by environment.
- Separate LLM and embedding model configuration.
- Treat external AI calls as failure-prone; implement retries and fallbacks.
- Capture token and latency telemetry for every invocation.

## 4. Required Platform Setup

### AWS Account and IAM
1. Create IAM principal for app runtime.
2. Grant minimum required Bedrock actions:
   - `bedrock:InvokeModel`
   - `bedrock:InvokeModelWithResponseStream`
   - `bedrock:Converse`
   - `bedrock:ConverseStream`
3. Restrict resources where possible.

### Model Access
1. Open Bedrock Model Access console.
2. Enable required LLM and embedding models per region.
3. Validate access with a smoke invocation.

### Region Strategy
- Preferred default region: `us-east-1` unless governance/latency requires another.
- Keep region explicit via `AWS_DEFAULT_REGION`.

## 5. Standard Environment Contract
Use this minimum env schema:

```env
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_SESSION_TOKEN=...               # optional
AWS_DEFAULT_REGION=us-east-1
BEDROCK_LLM_MODEL=us.meta.llama4-scout-17b-instruct-v1:0
BEDROCK_EMBEDDING_MODEL=amazon.titan-embed-text-v2:0
BEDROCK_ENDPOINT_URL=...            # optional
```

Security controls:
- `.env*` files must be ignored by git.
- Prefer role-based auth in cloud runtime over static keys.
- Rotate credentials and audit usage regularly.

## 6. Implementation Workflow

### Phase A: Client Factory
1. Build a shared `build_bedrock_client()`.
2. Accept explicit region and optional endpoint override.
3. Use explicit env credentials where provided; otherwise rely on AWS chain.

Exit criteria:
- One canonical client creation path used by all Bedrock calls.

### Phase B: LLM Inference via Converse API
1. Build message payload using Bedrock Converse schema.
2. Externalize `maxTokens`, `temperature`, `topP`.
3. Parse `response.output.message.content[0].text` safely.
4. Record `usage` and `metrics.latencyMs`.

Exit criteria:
- Deterministic text result extraction with telemetry attached.

### Phase C: Embeddings via InvokeModel
1. Build model-specific JSON payload (`inputText`).
2. Parse body stream reliably and extract `embedding` vector.
3. Validate vector dimensionality.

Exit criteria:
- Embeddings are reproducible and schema-validated.

### Phase D: Resilience and Cost Controls
1. Add bounded retries for transient failures.
2. Add timeout budgets per endpoint.
3. Enforce token ceilings by route/use case.
4. Route large workloads to async batch pipelines when possible.

Exit criteria:
- Calls stay within latency and cost budgets under normal load.

## 7. Decision Framework

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Text API | Converse API | InvokeModel text payloads | Use Converse for provider-agnostic chat semantics. |
| Embeddings API | InvokeModel | Converse (if model supports) | Use model-native embedding contract for reliability. |
| Auth in cloud | IAM role/STS | Static access keys | Use roles in production; keys only for local/dev bootstrap. |
| Model selection | Env-driven model ID | Hard-coded model ID | Always keep model ID configurable. |

## 8. Validation Strategy

### Functional Tests
- Smoke test for both LLM and embedding models.
- Parsing tests for expected response structure.
- Backward-compat tests if model versions change.

### Fault Injection Tests
- Simulate `AccessDeniedException`, timeout, throttling.
- Verify retry and fallback behavior.
- Verify clear operator-facing error messages.

### Security Validation
- Secret scanning in CI.
- Ensure no credentials in logs.
- Validate least-privilege IAM policies periodically.

## 9. Benchmarks and SLO Targets
- LLM request success rate (excluding caller cancellation): `>= 99.5%`
- Embedding request success rate: `>= 99.9%`
- LLM end-to-end latency p95 (medium prompt/output): `<= 4.0 s`
- Embedding latency p95 (single input): `<= 1.0 s`
- Budget drift from planned token usage: `<= 10%` per release cycle

## 10. Operational Runbook
- On `AccessDeniedException`:
  - Verify model access enabled in target region.
  - Verify IAM action and resource scope.
- On high latency:
  - Check region/model pairing and concurrency.
  - Reduce output token limits.
- On cost spike:
  - Inspect usage telemetry and route-level token ceilings.
  - Move heavy/background tasks to lower-cost models where quality permits.

## 11. Agent Execution Checklist
- [ ] Env contract implemented and documented.
- [ ] Shared Bedrock client factory in place.
- [ ] Converse and embedding APIs both covered.
- [ ] Retry, timeout, and error taxonomy implemented.
- [ ] Token/latency telemetry instrumented.
- [ ] Security and cost controls verified in CI and runtime.

## 12. Reuse Notes
This module is transportable across Python services. Adapt only:
- Model IDs by workload quality/cost profile.
- IAM scope by environment.
- Latency and cost SLO thresholds by product tier.
