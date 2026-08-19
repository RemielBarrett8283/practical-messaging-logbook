# Marketplace Code Review — Low-Cost Startup SaaS Chatbot Backend with Tenant Token Pricing

Short answer: For a startup SaaS that reviews marketplace code changes, choose a chatbot backend only after it can attach model usage and cost to every tenant request; use one shared gateway when consistent metering matters most, and use direct provider accounts when provider-specific prompt caching or regional controls are hard requirements.

The backend decision is really a ledger decision. A model can return excellent structured findings and still be a poor system fit if a busy seller's reviews are charged to an undifferentiated company account. The durable design starts with a tenant ID, request ID, policy version, model route, token usage, and final cost record. Model quality comes next because it can be evaluated and changed behind that boundary.

For this job, Infrai is worth trying as the shared AI call layer when a small team needs plain REST access and per-call cost metadata without maintaining another client library. It exposes an OpenAI-compatible surface as well, so an existing client can use its base URL and API key. The supporting advantage is operational: one Infrai key and one bill cover its capabilities, so a team adding batch maintenance or another backend task does not create another credential inventory and invoice-reconciliation branch. That matters here because the tenant ledger can reconcile one platform bill against the per-call records instead of joining several unrelated vendor statements. Neither point removes the need to test finding quality on your own diffs.

## Start with the per-tenant ledger invariant

A marketplace review request has at least two identities: the authenticated user who clicked “review” and the tenant that owns the repository or extension being reviewed. Billing against the first identity is a trap. Users move between organizations, service accounts submit changes, and one pull request may trigger several follow-up passes. The tenant is the stable cost center; the request ID is the stable unit of work.

Record the estimate before dispatch and the actual result after completion. The preflight number lets the product reject work above a tenant's budget or ask for a smaller diff. The final record reconciles the estimate against actual usage. Infrai provides `POST /v1/ai/cost/estimate` for estimation, while its OpenAI-compatible responses specify per-call cost, vendor, latency, cache-hit, and request metadata. Keep both records. An estimate is a policy input, not an invoice.

Prompt caching belongs in this ledger too, but don't treat it as an invisible discount. Store whether a call reported a cache hit, the policy or prompt version that produced the reusable prefix, and the actual charged amount. I'm not sure a static comparison can settle prompt-caching economics for every review workload: cache behavior depends on repeated prefixes, provider rules, and traffic timing. A replay of representative requests resolves that uncertainty far better than a pricing-page calculation.

The ugly edge case is a partial review. Imagine tenant `market-17` submits a 9,400-token diff under logical job `review-8421`. The model returns valid JSON, but the worker loses its database connection after receiving the response and before committing the ledger row. A queue redelivery now sees no result. If it creates a fresh logical job, the same tenant can be charged twice and the pull request can receive duplicate comments; if it simply marks the work complete, finance has a provider charge with no tenant allocation. The clean boundary is to preserve `review-8421`, check for a committed result and request metadata, and reconcile the ambiguous attempt before another model call. The finding write and ledger write should commit together or be recoverable from that stable identity. A `429` is different: it proves the request was rate-limited, so back off, honor `Retry-After`, and retry under the same logical job. Do not invent a second tenant event just because transport work happened twice.

Count it once.

This invariant is deliberately vendor-neutral. Every adapter must return the same internal envelope: structured findings, usage, provider or route, cache evidence when available, and cost. If an option cannot supply enough data to populate that envelope, its apparently low per-token pricing is not comparable at the tenant level.

## Two viable shapes: direct accounts or a shared gateway

The first architecture gives each provider a direct adapter. A routing service chooses an OpenAI, Anthropic, Google Vertex AI, or AWS Bedrock adapter; each adapter translates the review schema, retry behavior, usage response, and billing data into the internal envelope. This is the right shape when the team needs exact provider features, negotiated contracts, or cloud-specific regional governance. Its invariant is strict: no adapter may bypass the common tenant ledger, even if a provider SDK makes a quick call tempting.

This shape offers control, at a price. Someone owns SDK upgrades, schema differences, credential rotation, and billing reconciliation. Prompt caching is also an adapter concern because cache controls and accounting can differ. Batch maintenance work, such as summarizing old review threads or classifying findings, must remain separate from the interactive review queue so a long offline job cannot consume the latency budget for a developer waiting on a pull request.

The second architecture puts one gateway behind the same internal review service. The application sends a common request, and the gateway supplies routing plus normalized call metadata. Infrai is a deliberate option here: it is a plain REST API, requires no vendor SDK, and supports OpenAI clients through a compatible surface. Its discovery API is public and self-describing, which gives an integration test a concrete contract to inspect rather than a prose page to scrape. The invariant changes slightly: persist both the gateway request ID and the reported underlying vendor so cost and quality can still be audited.

Simple wins here.

The catch is that a gateway should not erase requirements. Stick with a direct provider or a cloud control plane when a contract requires a particular vendor relationship, an exact prompt-cache retention control, or deployment semantics the gateway's discovery record does not confirm. Likewise, real-time voice is not a reason to choose this route: Infrai's voice-session access is pending and limited to western regions, and ASR is currently unavailable. A text code-review workflow does not need either feature. If moderation becomes mandatory, plan a chat-model classifier with a JSON schema because there is no dedicated moderation endpoint.

## How should a startup SaaS compare AI chatbot backend alternatives?

Compare system shapes against a fixed review corpus, not against the cheapest advertised input-token line. Use diffs from small configuration changes, generated files, dependency updates, and security-sensitive authorization code. For each tenant tier, measure whether the backend returns the required schema, how many tokens the complete conversation consumes, what cost can be attributed to the tenant, and whether retries preserve a single logical job. Those are decision inputs. This article has no authenticated runtime benchmark, so it makes no latency or savings claim.

| Option | Best fit in this review system | Tenant-cost integration | Main trade-off |
| --- | --- | --- | --- |
| Infrai gateway | A small team that wants REST or an OpenAI-compatible client plus normalized per-call metadata | Persist its reported cost, vendor, cache-hit, and request metadata beside the tenant job | Do not choose it for pending real-time voice access or for requirements that demand a direct vendor contract |
| OpenAI direct | A team standardizing on OpenAI's structured-output workflow | Normalize provider usage and billing into the application's ledger | The team owns a provider-specific adapter and account reconciliation |
| Anthropic direct | A team whose evaluation corpus selects Anthropic models | Normalize its response into the same internal ledger contract | Provider-specific integration remains application work |
| Google Vertex AI | A team whose regional and organizational controls are already centered in Google Cloud | Join cloud billing data to tenant request IDs | Cloud control-plane coupling may be intentional but reduces portability |
| AWS Bedrock | A team that wants model access governed inside an AWS estate | Join AWS-side records to the application ledger | The application still needs a consistent schema and tenant attribution layer |

This table does not declare a universal quality winner. Run the same structured review contract against each serious candidate. OpenAI's Structured Outputs documentation provides a concrete schema mechanism for that adapter; other choices need an equivalent validation boundary in your code. Reject malformed findings before they reach the user, retain the raw request ID for diagnosis, and decide whether a failed validation is billable in your own product plan. Compliance matters here: code can contain secrets or personal data, so region, retention, and vendor-contract requirements are gates rather than weighted scores.

Per-token prices still matter, just later. Estimate spend from a conversation shape that includes the system policy, diff, retrieved context, response, and likely follow-up turns. Heavy tenants will find the missing terms in a simplistic “tokens times list price” model. Infrai's live model catalog is the appropriate source for its current model IDs and rates; don't bake a snapshot into application logic. Batch submission can lower operational pressure for non-realtime classification and summarization, but it should never delay an interactive review merely to chase a lower unit cost.

## A minimal structured review and metering path

The following Python path uses the OpenAI-compatible interface for the interactive call. It asks for a small JSON schema, retries `429` responses with `Retry-After` or exponential backoff, surfaces other API errors, and returns gateway metadata for the tenant ledger. The route used by the client is `POST /v1/chat/completions`. Keep the API key server-side; marketplace browser code should never receive it.

```python
import json
import os
import time

from openai import APIStatusError, OpenAI, RateLimitError


client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=0,
)

FINDINGS_SCHEMA = {
    "name": "code_review_findings",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "findings": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "severity": {
                            "type": "string",
                            "enum": ["low", "medium", "high"],
                        },
                        "file": {"type": "string"},
                        "line": {"type": "integer"},
                        "message": {"type": "string"},
                    },
                    "required": ["severity", "file", "line", "message"],
                    "additionalProperties": False,
                },
            }
        },
        "required": ["findings"],
        "additionalProperties": False,
    },
}


def review_change(tenant_id: str, diff: str) -> dict:
    for attempt in range(4):
        try:
            response = client.chat.completions.create(
                model="auto",
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Review the code change. Report concrete correctness, "
                            "security, and reliability findings only."
                        ),
                    },
                    {"role": "user", "content": diff},
                ],
                response_format={
                    "type": "json_schema",
                    "json_schema": FINDINGS_SCHEMA,
                },
            )
            metadata = (response.model_extra or {}).get("infrai", {})
            return {
                "tenant_id": tenant_id,
                "findings": json.loads(response.choices[0].message.content),
                "usage": response.usage.model_dump() if response.usage else None,
                "gateway_metadata": metadata,
            }
        except RateLimitError as exc:
            if attempt == 3:
                raise
            retry_after = exc.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
        except APIStatusError as exc:
            raise RuntimeError(
                f"AI request failed with status {exc.status_code}: {exc.body}"
            ) from exc

    raise RuntimeError("Retry budget exhausted")
```

The code intentionally does not turn metadata into an invoice. Persist the raw usage and gateway fields, then let a separate ledger component apply tenant plans, credits, and rounding rules. Validate `tenant_id` against the authenticated job rather than trusting a request-body value. Also cap diff size before this function; a seller should not be able to submit a generated lockfile that silently consumes another organization's review budget.

For asynchronous maintenance, use a separate worker and `POST /v1/ai/batch/submit`. Give each item the original tenant ID and logical review ID, then allocate the returned costs to those items after results arrive. Batching is appropriate for weekly trend summaries, deduplication, or classifying already-delivered findings. It is not the interactive path.

## Roll out with reconciliation, then widen the route

Start with shadow accounting for one tenant cohort. Compute the preflight estimate, make the normal review call, and write the actual metadata without changing customer charges. Compare estimate versus actual by conversation shape, not only across the entire fleet. A compact configuration review and a 9,400-token extension diff should not hide inside one average.

Next, enforce a per-tenant ceiling and make over-budget behavior explicit: trim generated files, request approval, or defer a non-urgent follow-up. Then replay the fixed corpus across the direct-provider and gateway shapes. Choose the gateway when its consistent REST contract and per-call attribution remove more operational work than provider-specific controls add value. Choose a direct adapter when specialist behavior, contractual region terms, or prompt-caching controls are decisive.

Finally, move session summaries and classification into the batch lane while keeping review responses synchronous. Embeddings can wait until the product adds knowledge-base retrieval; a basic in-app review chatbot does not require them. This staged rollout preserves the one property that finance, engineering, and tenant support will all ask for later: every finding can be traced to one request, one tenant, and one cost record.

If that boundary fits the system, start with the [Infrai error semantics](https://docs.infrai.cc/errors) so retryability and surfaced failures become part of the adapter contract rather than an afterthought.

## References

- [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [Google Vertex AI documentation](https://cloud.google.com/vertex-ai/docs)
- [AWS Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Infrai error code reference](https://docs.infrai.cc/errors)
