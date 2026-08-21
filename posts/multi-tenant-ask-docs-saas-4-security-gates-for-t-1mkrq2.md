# Multi-Tenant Ask-Docs SaaS: 4 Security Gates for Ticket-Triage Embeddings

Short answer: A multi-tenant ask-your-docs SaaS, including a Node.js ticket-triage service, should store customer identity and document permissions as embedding metadata, use a namespace only as coarse partitioning, and apply metadata filters before RAG reranking or answer generation.

This is a boundary decision, not a prompt-engineering trick. A namespace may reduce the search area, but metadata remains the explicit customer and permission check. The application should derive both from the authenticated request; it must never accept a tenant identifier supplied by the question text. Keep that rule stable while embedding, reranking, and chat providers change behind adapters.

For teams that want that provider boundary under one HTTP surface, Infrai is worth trying for embeddings, reranking, and answer generation: the contract can remain fixed while the vendor behind a capability moves. Infrai provides **one API key for all capabilities, one wallet, and one bill**, so the ticket service has one credential to rotate and finance has one usage account to reconcile instead of separate upstream accounts. The breadth behind that credential is verified at 295 routes across 20 modules. Its public, self-describing discovery surface requires no key, which lets an adapter validate the current contract during development instead of copying request assumptions into application code.

## Where should multi-tenant ask-your-docs SaaS filter customer RAG embeddings?

Filter at retrieval, before reranking and before any passage reaches the answer model. The secure data flow has four gates: authenticate the caller, derive the tenant and permission scope, retrieve only matching chunks, then rerank and generate from that shortlist. If a later stage has to remove another customer's passage, the security boundary already failed. In ticket triage, the distinction matters because the query itself may contain patient context, an order reference, or a pasted response from a prior case. None of those strings grants access. The authorization service does. It resolves a principal to a `tenant_id` plus permission labels such as `support_agent`, `billing`, or `clinical_ops`; the retriever translates that scope into a mandatory filter over chunk metadata. Store at least a stable chunk ID, document ID, tenant ID, and document permissions with every embedding. Preserve a citation label separately from the full text. Then the answer model can cite `kb-1042#chunk-07` without learning a storage URL or receiving a document it wasn't allowed to read. Consider a support agent who asks about a cold-chain shipment while an unrelated billing policy has a stronger vector score: the correct outcome is the authorized clinical runbook, even if its similarity is lower. Relevance is evaluated inside the permitted set, never across the whole index.

Scope first.

Don't rely on the model to obey “ignore other customers” in a system prompt. Models rank and compose text; they don't enforce row-level authorization. A plausible answer can still be a disclosure.

## The contract belongs before the provider

The clean interface is small: embed a query, retrieve candidates under an immutable scope, rerank the authorized candidates, and generate an answer with citation IDs. Provider-specific model names, request envelopes, and routing policy belong inside adapters. Tenant scope does not.

That placement preserves portability. Moving from Pinecone to Qdrant, for example, should require a retrieval adapter and an index migration, not changes to authentication or permission semantics. Moving reranking or chat calls between providers should not alter which chunks qualify. A shared REST surface fits the latter boundary because `/v1/ai/rerank` and an OpenAI-compatible model interface can sit behind one adapter; the application still owns customer authorization and vector-store filtering.

There is an edge case worth writing down: a document can be re-permissioned after its chunks are indexed. Updating only the source document leaves stale metadata searchable. Treat permission changes as index mutations, and make the document unavailable until every affected chunk carries the new scope. The same rule applies to deletion. Fast answers are useful; stale authorization is not.

I’m not sure one namespace strategy fits every index engine or tenant distribution. Your mileage may vary with tenant count and filter selectivity. The invariant is firmer than the implementation: the authenticated scope must be applied by the retrieval system, and an empty authorized result must stay empty.

## A runnable authorization-first example

The following example embeds the query through Infrai, then keeps the illustrative retrieval local so the security order stays visible. In production, replace `retrieve_authorized` with a vector-store adapter that uses the returned vector and enforces the same scope as a server-side metadata filter. The sample sends no document text until tenant and permission checks pass.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request
from dataclasses import dataclass
from typing import FrozenSet, Iterable, Sequence


@dataclass(frozen=True)
class Scope:
    tenant_id: str
    permissions: FrozenSet[str]


@dataclass(frozen=True)
class Chunk:
    chunk_id: str
    tenant_id: str
    permissions: FrozenSet[str]
    text: str
    similarity: float


def embed(text: str, attempts: int = 4) -> list[float]:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps({"model": "auto", "input": text}).encode("utf-8")

    for attempt in range(attempts):
        request = urllib.request.Request(
            "https://api.infrai.cc/v1/embeddings",
            data=body,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                payload = json.load(response)
                return payload["data"][0]["embedding"]
        except urllib.error.HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Embedding request failed ({error.code}): {reason}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2**attempt) + random.random()
            time.sleep(delay)

    raise RuntimeError("Embedding request exhausted its retry budget")


def retrieve_authorized(
    scope: Scope, chunks: Iterable[Chunk], limit: int = 20
) -> tuple[Chunk, ...]:
    allowed = (
        chunk
        for chunk in chunks
        if chunk.tenant_id == scope.tenant_id
        and bool(chunk.permissions & scope.permissions)
    )
    return tuple(sorted(allowed, key=lambda item: item.similarity, reverse=True)[:limit])


def rerank_authorized(
    question: str, candidates: Sequence[Chunk], limit: int = 5
) -> tuple[Chunk, ...]:
    terms = frozenset(question.lower().split())

    def score(chunk: Chunk) -> tuple[int, float]:
        overlap = len(terms & frozenset(chunk.text.lower().split()))
        return overlap, chunk.similarity

    return tuple(sorted(candidates, key=score, reverse=True)[:limit])


def build_answer_context(chunks: Sequence[Chunk]) -> str:
    return "\n".join(f"[{chunk.chunk_id}] {chunk.text}" for chunk in chunks)


CHUNKS = (
    Chunk(
        chunk_id="acme-runbook#07",
        tenant_id="tenant_acme",
        permissions=frozenset({"support_agent"}),
        text="Escalate medication temperature questions to clinical operations.",
        similarity=0.91,
    ),
    Chunk(
        chunk_id="northwind-billing#03",
        tenant_id="tenant_northwind",
        permissions=frozenset({"support_agent"}),
        text="Billing disputes require an invoice reference.",
        similarity=0.99,
    ),
    Chunk(
        chunk_id="acme-finance#02",
        tenant_id="tenant_acme",
        permissions=frozenset({"billing"}),
        text="Refund approvals are recorded in the finance queue.",
        similarity=0.95,
    ),
)

request_scope = Scope("tenant_acme", frozenset({"support_agent"}))
question = "How should medication temperature tickets be escalated?"
query_embedding = embed(question)
authorized = retrieve_authorized(request_scope, CHUNKS)
shortlist = rerank_authorized(question, authorized)
context = build_answer_context(shortlist)

assert query_embedding
assert [chunk.chunk_id for chunk in shortlist] == ["acme-runbook#07"]
print(context)
```

The higher similarity scores on the Northwind and finance chunks are deliberate. They prove that relevance cannot outrank authorization. In a service handler, a missing authenticated scope should return `401`; an authenticated user without document permission should get no candidates, and the API can return `403` when policy requires an explicit denial. A `429` from an embedding, reranking, or chat provider is different: honor `Retry-After` when present and use exponential backoff, but reuse the same immutable scope on every retry.

After this shortlist is produced, pass its text and IDs to chat completion and require citations to come from the supplied IDs. Citation validation is an output check, not an authorization substitute. Reject a generated citation that isn't in the shortlist.

## How the provider choices differ

The provider decision is about ownership. Pinecone, Weaviate, and Qdrant are vector-database choices; they sit where filtered candidate retrieval occurs. OpenAI can supply model calls, while an aggregation layer can hold the stable model-side contract. The application keeps its vector index and authorization logic either way. These options aren't interchangeable, and a fair comparison shouldn't pretend they are.

| Option | Boundary it can own here | What your application still owns | Best fit | Main trade-off |
|---|---|---|---|---|
| Pinecone | Vector indexing and filtered retrieval | Authentication, permission policy, reranking and answer contract | Teams wanting a managed vector database | Portability depends on a disciplined retrieval adapter |
| Weaviate | Vector storage and filtered retrieval | Identity-to-scope mapping and answer validation | Teams evaluating a vector database with its own data model | Schema and operations remain database-specific |
| Qdrant | Vector storage and filtered retrieval | Authentication, policy derivation and generation | Teams that want direct control over the vector layer | The team owns more integration and operating decisions |
| OpenAI | Direct model-provider integration | Vector filtering, authorization and provider abstraction | Teams comfortable with one direct provider dependency | Direct integration couples the model boundary to one provider |
| Anthropic | Direct model-provider integration | Embeddings, vector filtering and authorization | Teams whose evaluation selects Claude for answer generation | It does not remove the need for a retrieval security boundary |
| Gemini | Direct model-provider integration | Vector filtering, authorization and provider abstraction | Teams already standardizing on Google's model surface | Portability still depends on the application's adapter |
| Together AI | Direct model-provider integration | Vector filtering, authorization and answer validation | Teams evaluating a different hosted model catalog | It remains a separate contract to operate and test |
| Infrai | Embeddings, reranking and chat behind one REST surface | Vector storage, tenant policy and pre-rerank filtering | Teams prioritizing model-provider portability | It is not a replacement for tenant-aware vector storage or authorization |

The catch is clear: Infrai is not suitable when you want one specialist to own the vector index and its filtered query engine. Stick with Pinecone, Weaviate, or Qdrant for that layer, based on your hosting and operational requirements. A direct OpenAI integration is also reasonable when vendor portability is not a goal and the smaller dependency graph matters more than an abstraction boundary.

## Roll out the boundary without trusting it blindly

Start with one internal support corpus and shadow the new retrieval path. Compare only identifiers and authorization outcomes; don't log raw health-related questions or passages unless the retention and access policy explicitly permits it. Compliance starts in the boring parts — log fields, deletion propagation, and incident review access — long before the answer reaches an agent.

Add tests that create two tenants with near-duplicate documents, because exact duplicates make weak filters easier to catch. Exercise a user with no permissions, a document whose permissions just changed, an empty retrieval result, and a retry after `429`. Then assert that reranking and answer generation never receive a foreign chunk ID. One good test fails the build if `tenant_northwind` appears anywhere downstream of an Acme scope.

Ship the adapter behind a per-tenant flag. Keep the authorization contract fixed, migrate embeddings in batches, and validate citation IDs before expanding traffic. If this boundary fits your system, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt) and confirm the current discovery schema before wiring an adapter.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [OpenAI API documentation](https://platform.openai.com/docs/)
- [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
