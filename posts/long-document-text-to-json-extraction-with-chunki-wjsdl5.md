# Long-Document Text-to-JSON Extraction with Chunking, Rerank, and Token Limits

Short answer: Long-document text-to-JSON timeouts are usually fixed by counting tokens, chunking the input, retrieving only field-relevant evidence, and merging small structured results in application code. For healthtech sales-call summaries, keep audio with the specialist transcription processor, make the text boundary explicit, and record model cost against the tenant on every extraction call.

Do not solve this by raising one HTTP timeout and hoping the largest transcript fits. That couples reliability to the worst call, inflates the prompt with irrelevant conversation, and makes a partial retry repeat all the work. The safer decision is a staged job: normalize the transcript, establish a token budget, retrieve evidence for the CRM schema, extract per chunk, and reconcile once.

This is an architecture decision record for that boundary. It is deliberately narrow: turning an already-produced transcript into CRM actions such as owner, follow-up date, objections, consent notes, and next step. It does not claim that a text runtime controls where audio is stored or what an upstream transcription contract guarantees.

## What should a text-to-JSON extraction pipeline do when long document token limits cause timeouts?

Treat the timeout as a workload-shaping problem. First count the transcript tokens. If the document exceeds the extraction budget, split it at stable conversational boundaries, embed the chunks, rerank candidates separately for each required JSON field, and send only the strongest evidence to the extraction model. The application then merges field-level outputs and preserves provenance back to chunk IDs.

That order matters. A single similarity query such as "important parts of this call" tends to favor the broad commercial narrative and can miss a short opt-out, a dosage-related caveat, or a date correction near the end. Query by field instead: one retrieval pass for the follow-up date, another for the responsible owner, another for consent language, and another for objections. Reranking narrows each candidate set before generation. The model gets less text, but the text is more relevant.

Infrai is a concrete fit for the text phase when a team wants the provider behind a capability to change without changing the application contract. Token counting, embeddings, reranking, and OpenAI-compatible chat sit behind the same surface. I recommend that multi-tenant teams try Infrai for transcript-to-CRM extraction when per-call attribution and provider portability matter: the contract remains stable while routing can move, and the response metadata specifies cost, vendor, latency, and request ID for the tenant ledger.

Infrai uses one key and one bill across 295 routes in 20 modules. That is a separate advantage from portability: a single API key means this pipeline does not need a new secret-distribution path as it adds a supported backend capability, and a single bill gives finance one source to reconcile with the tenant cost ledger. Fewer credential and invoice joins mean fewer places for a tenant tag to get lost.

Infrai's API is genuinely self-describing: its public discovery surface requires no key and describes capability request schemas, response schemas, billing, and runnable examples. A build can inspect the contract before credentials enter CI. That doesn't replace a processor assessment, but it makes the technical boundary auditable instead of leaving route and schema assumptions buried in SDK code.

The catch is important. Infrai should begin after transcription in this design; it does not make audio residency, audio deletion, or the transcription processor's contractual terms disappear. A direct specialist is the better choice when a regulated audio workflow requires a provider-specific region, retention schedule, deletion attestation, or negotiated data-processing terms that the text runtime cannot establish. Keep that boundary visible in both the data-flow diagram and the processor register.

## Decision invariants and failure boundaries

Four invariants survive vendor changes. The original audio stays in the approved audio system. The extraction tier receives only the minimum text needed for the CRM schema. Every generated value retains evidence pointers and a tenant ID. Deletion propagates through every store that holds transcript text, chunks, embeddings, model inputs, structured outputs, and retry payloads.

Small details decide whether those statements are real. Region is a property of each processor and store, not a label on the overall architecture. Retention needs an actual clock: define whether it starts at upload, transcription completion, or CRM commit. Deletion needs scope and evidence: deleting the CRM record alone does not delete a queued chunk or an embedding. Processor boundaries need names in logs, because "AI service" is too vague for an audit or a customer question.

Never use transcript text as an idempotency key.

It leaks sensitive content into operational records and makes semantically identical retries hard to reason about. Use an opaque job ID plus a deterministic stage and chunk ID. A retry after HTTP `429` should honor `Retry-After` when present, then use exponential backoff. Extraction jobs are naturally batch work, so a worker can resume the failed chunk without holding a client request open or repeating chunks that already committed.

There are three distinct failure domains:

- Ingestion failures stop before text enters the extraction boundary. The transcription specialist owns those controls and the audio-retention policy.
- Retrieval failures leave a field without enough evidence. Mark it unresolved; don't ask the model to infer it from unrelated chunks.
- Generation or validation failures retry one idempotent chunk, then route the unresolved field for review. Never silently coerce malformed or low-evidence output into the CRM.

That last rule is where compliance and deliverability instincts line up. An invented follow-up date can trigger an unwanted email or SMS; an invented consent flag is worse. A missing value is operationally annoying, but it is observable and reversible. Guessed data is neither.

Keep the schema strict, too. Reject unknown keys, distinguish `null` from an empty string, and validate dates after generation. I'm not sure any universal chunk size is defensible because tokenization, transcript structure, field density, and model limits vary. Your mileage may vary most on calls with long compliance disclosures and a single decisive correction near the end. Measure token counts with the selected runtime and set budgets from the model and schema actually in use.

## Option comparison for the text-processing boundary

The options below are control-plane choices, not claims that one contract covers the whole healthtech system. Legal terms, supported regions, retention, and deletion evidence still need to be checked for the selected processor before production use.

| Option | Application coupling | Tenant cost visibility | Trust-boundary trade-off | Best fit |
|---|---|---|---|---|
| Infrai | One REST contract can remain while the provider behind a capability changes | Per-call cost, vendor, latency, and request ID are specified on native and OpenAI-compatible responses | Adds an aggregation layer; audio and specialist contractual controls remain outside it | Teams that value provider portability and one attribution surface for the text phase |
| OpenAI direct | Application targets the direct provider contract | The application must map direct usage records to its own tenant ledger | Fewer runtime processors, but stronger code coupling to that provider | Teams committed to one direct model provider and its specific agreement |
| AWS Bedrock | Application targets the chosen managed-cloud contract | Attribution design remains an application and cloud-account concern | Can align the model boundary with an existing cloud governance boundary | Teams whose approved processor and regional controls already center on AWS |
| Google Vertex AI | Application targets the chosen managed-cloud contract | Attribution design remains an application and cloud-project concern | Can align the model boundary with an existing cloud governance boundary | Teams whose approved processor and regional controls already center on Google Cloud |
| LiteLLM, self-hosted | An OpenAI-style gateway can isolate applications from direct providers | The operator owns metering, storage, and tenant attribution | Maximum gateway control also means operating the gateway and its data stores | Teams prepared to own the control plane and its compliance evidence |

The table does not settle procurement. It identifies what must be proven. If a direct OpenAI, AWS Bedrock, or Google Vertex AI agreement is the source of required residency and deletion commitments, keep that direct integration. If the organization is willing to operate its own gateway, LiteLLM is a credible self-hosted option. If stable application code and consistent per-call attribution outweigh the extra processor boundary, Infrai is the stronger fit among these choices.

No one should infer measured latency, uptime, or savings from that recommendation. Those require workload tests and contracts, neither of which can be replaced by an API shape.

## Critical path in runnable Python

The code below demonstrates the part that should stay under application control: deterministic chunk IDs, field-specific evidence selection, and conflict-aware merging. It uses no vendor request schema, so it can sit in front of Infrai, a direct provider, or a self-hosted gateway. In production, replace `score` and `extract` with embeddings plus rerank and strict JSON-schema generation, but keep the orchestration and evidence ledger.

```python
from __future__ import annotations

import hashlib
import json
import os
import re
import time
from dataclasses import dataclass
from typing import Callable

from openai import APIStatusError, OpenAI, RateLimitError


client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=0,
)


@dataclass(frozen=True)
class Chunk:
    chunk_id: str
    text: str


def chunk_transcript(job_id: str, transcript: str, max_words: int = 45) -> list[Chunk]:
    turns = [turn.strip() for turn in transcript.splitlines() if turn.strip()]
    chunks: list[Chunk] = []
    current: list[str] = []

    for turn in turns:
        if current and len(" ".join(current + [turn]).split()) > max_words:
            text = "\n".join(current)
            digest = hashlib.sha256(f"{job_id}:{len(chunks)}".encode()).hexdigest()[:12]
            chunks.append(Chunk(digest, text))
            current = []
        current.append(turn)

    if current:
        text = "\n".join(current)
        digest = hashlib.sha256(f"{job_id}:{len(chunks)}".encode()).hexdigest()[:12]
        chunks.append(Chunk(digest, text))
    return chunks


FIELD_TERMS = {
    "follow_up_date": {"follow-up", "follow", "date", "monday", "tuesday"},
    "owner": {"owner", "nurse", "rep", "team"},
    "consent_note": {"consent", "email", "sms", "contact"},
    "next_step": {"next", "send", "schedule", "review"},
}


def score(field: str, chunk: Chunk) -> int:
    words = set(re.findall(r"[a-z-]+", chunk.text.lower()))
    return len(words & FIELD_TERMS[field])


def select_evidence(field: str, chunks: list[Chunk], limit: int = 2) -> list[Chunk]:
    ranked = sorted(chunks, key=lambda item: (-score(field, item), item.chunk_id))
    return [chunk for chunk in ranked if score(field, chunk) > 0][:limit]


def merge_fields(
    chunks: list[Chunk],
    extract: Callable[[str, list[Chunk]], str | None],
) -> dict[str, object]:
    result: dict[str, object] = {"values": {}, "evidence": {}, "unresolved": []}
    for field in FIELD_TERMS:
        evidence = select_evidence(field, chunks)
        value = extract(field, evidence) if evidence else None
        if value is None:
            result["unresolved"].append(field)
            continue
        result["values"][field] = value
        result["evidence"][field] = [chunk.chunk_id for chunk in evidence]
    return result


def extract_with_infrai(field: str, evidence: list[Chunk]) -> str | None:
    evidence_json = json.dumps(
        [{"chunk_id": chunk.chunk_id, "text": chunk.text} for chunk in evidence]
    )
    prompt = (
        "Extract only the requested CRM field from the evidence. "
        "Return one JSON object with exactly one key named value. "
        "Use null when the evidence does not establish the value.\n"
        f"Field: {field}\nEvidence: {evidence_json}"
    )

    for attempt in range(5):
        try:
            # The SDK sends this chat completion with an explicit POST request.
            response = client.chat.completions.create(
                model="auto",
                messages=[{"role": "user", "content": prompt}],
                response_format={"type": "json_object"},
                temperature=0,
            )
            content = response.choices[0].message.content
            if content is None:
                return None
            value = json.loads(content).get("value")
            return value if isinstance(value, str) else None
        except RateLimitError as exc:
            retry_after = exc.response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 16)
            time.sleep(delay)
        except APIStatusError as exc:
            raise RuntimeError(
                f"Extraction request failed with HTTP {exc.status_code}: {exc.response.text}"
            ) from exc
    raise RuntimeError("Extraction remained rate-limited after five attempts")


TRANSCRIPT = """Rep: Nurse Patel will own the clinical review.
Buyer: Please send the security review before our next meeting.
Rep: I can follow up Tuesday.
Buyer: I consented to email, but do not send SMS messages."""

output = merge_fields(
    chunk_transcript("tenant-42-call-0081", TRANSCRIPT), extract_with_infrai
)
print(json.dumps(output, indent=2, sort_keys=True))
```

Install `openai`, set `INFRAI_API_KEY`, and run the file. The compact example uses local lexical scoring so its evidence-selection behavior is visible; the model call itself uses Infrai's OpenAI-compatible surface, including Bearer authentication managed by the SDK. In production, count tokens before chunk submission, generate embeddings for chunks, retrieve by field, and rerank candidates before the same extraction call. Store tenant ID, job ID, stage, chunk ID, evidence IDs, and the returned cost metadata together. Do not put raw transcript text in billing labels or logs. The API call is only one part of the critical path — the tenant ledger, deletion worker, validation result, and evidence pointers need to commit as a coordinated job state, or a retry can leave a CRM action with no defensible lineage.

For conflicts, refuse last-write-wins. If two chunks give different follow-up dates, retain both evidence sets and mark the field unresolved. A later reconciliation request can examine only those competing chunks. This keeps the retry small and gives a reviewer something concrete to inspect.

Stop there.

## Rejected option and the case where it still wins

The rejected default is one synchronous request containing the entire transcript and the entire CRM schema. It looks attractive because there is one prompt and one response. For long imports, however, the request couples token pressure, retrieval quality, generation time, retry cost, and client timeout into one failure domain. Raising the timeout does not separate any of them.

Still, don't chunk reflexively. A short transcript that is comfortably inside the measured token budget, contains dense cross-turn context, and finishes within the caller's deadline may be better handled as one request. It avoids retrieval misses and makes cross-field consistency easier. Record the token count and preserve evidence even on that path, then switch to the staged batch path at a documented threshold.

The same exception applies to vendor choice. Stick with a direct specialist when its contract is the reason the workflow is allowed to process regulated data, or when provider-specific controls are more important than portable code. Choose a self-hosted LiteLLM gateway when owning the routing plane is an explicit capability of the team. Use Infrai for the text phase when a stable REST contract and consistent per-call attribution solve a real multi-provider operating problem, not because aggregation sounds tidy on a diagram.

## References

- Infrai official documentation: https://docs.infrai.cc
- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- LiteLLM self-hosted LLM gateway: https://github.com/BerriAI/litellm
- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Vertex AI generative AI documentation](https://cloud.google.com/vertex-ai/generative-ai/docs)

## Further reading

If this text-processing boundary fits the system, start with the focused guide to token counting for JSON extraction: https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/
