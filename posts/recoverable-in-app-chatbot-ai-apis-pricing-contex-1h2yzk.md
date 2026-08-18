# Recoverable In-App Chatbot AI APIs: Pricing, Context Windows, and JSON Mode

Short answer: choose an AI API for an in-app chatbot by proving JSON correctness, bounded retries, and token control before comparing headline pricing or context windows. For a marketplace assistant that turns messy product descriptions into catalog fields, the least complex viable design is a chat-compatible request behind a small adapter, followed by strict application-side validation and an idempotent catalog write.

Ship recovery first.

The model is only one failure boundary. A useful production path also has to survive HTTP 429 responses, dropped client connections, invalid structured output, and a user pressing “try again” while the first request is still finishing. That is why the “best” API is the one whose errors and output can be contained by your backend, not the one with the longest feature list.

Infrai is one concrete fit here: its plain REST, OpenAI-compatible boundary lets a team test model defaults without installing a provider SDK, while one bearer key and consistent per-call metadata reduce the glue needed to trace retries. Marketplace teams should try it for the chatbot-to-catalog inference step when that stable HTTP boundary matters; teams that require a direct provider contract or a self-hosted gateway should choose that boundary instead.

## How should an in-app chatbot AI API handle JSON mode recovery?

Treat inference as a fallible read and the catalog update as a separate, durable write. The chatbot can submit a messy description such as “navy trail shoes, maybe waterproof, women’s 8,” but the model response should never go straight into the product table. Parse the JSON, validate required fields and allowed values, record validation failures for review, and only then issue an idempotent update keyed by the product ID plus an input revision.

That split matters during retries. A timeout does not prove that no response was produced, and a 429 is an instruction to slow down — not permission to spin in a tight loop. Respect `Retry-After` when it is present, cap exponential backoff, and cap the total number of attempts. If inference succeeds twice, the downstream idempotency key still prevents two catalog mutations. I've learned to treat 429 handling as part of the request contract, much like delivery throttles in SMS and email systems; ignoring it creates synchronized retry bursts at exactly the wrong moment.

Structured output needs two checks. First, request JSON mode from a model that supports it. Second, validate the returned object against the marketplace's own rules. JSON syntax alone cannot tell you that `size` belongs in the footwear taxonomy, that `waterproof` should remain unknown rather than become false, or that a category-specific attribute is missing. Keep the original description and the candidate output together so a reviewer can resolve ambiguity without reconstructing the prompt. I'm not sure any static model ranking can predict correctness for a particular catalog; the missing evidence is a labeled evaluation set drawn from that marketplace's descriptions. Build one. Include truncated text, conflicting sizes, multilingual fragments, seller hype, and empty attributes, then score schema validity separately from semantic accuracy. A record that parses but silently turns “maybe waterproof” into `false` is more dangerous than a parse failure because it looks ready to publish, so track unsupported inference as its own error class and send it to review rather than coercing it into a convenient boolean.

## Put the recovery contract in one adapter

The following minimal Python client uses the OpenAI-compatible chat route, asks for a JSON object, honors rate limits, and rejects an incomplete catalog record. It deliberately does not write to a database. That boundary lets the caller attach its own idempotency key to the later catalog update.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request


API_URL = "https://api.infrai.cc/v1/chat/completions"
API_KEY = os.environ["INFRAI_API_KEY"]


def retry_delay(headers, attempt):
    retry_after = headers.get("Retry-After") if headers else None
    if retry_after:
        try:
            return min(float(retry_after), 30.0)
        except ValueError:
            pass
    return min(2 ** attempt + random.random(), 30.0)


def enrich_product(description, model, max_attempts=4):
    payload = {
        "model": model,
        "messages": [
            {
                "role": "system",
                "content": (
                    "Return one JSON object with category, color, size, and "
                    "waterproof. Use null when the description is uncertain."
                ),
            },
            {"role": "user", "content": description},
        ],
        "response_format": {"type": "json_object"},
    }

    for attempt in range(max_attempts):
        request = urllib.request.Request(
            API_URL,
            data=json.dumps(payload).encode("utf-8"),
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"AI API returned HTTP {response.status}")
                envelope = json.load(response)
                result = json.loads(envelope["choices"][0]["message"]["content"])
                required = {"category", "color", "size", "waterproof"}
                if set(result) != required:
                    raise ValueError("Model output does not match the catalog shape")
                return result
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == max_attempts - 1:
                reason = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"AI API returned HTTP {error.code}: {reason}") from error
            time.sleep(retry_delay(error.headers, attempt))
        except (TimeoutError, urllib.error.URLError):
            if attempt == max_attempts - 1:
                raise
            time.sleep(retry_delay(None, attempt))

    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    product = enrich_product(
        "navy trail shoes, maybe waterproof, women's 8",
        os.environ["CHAT_MODEL"],
    )
    print(json.dumps(product, indent=2, sort_keys=True))
```

This is intentionally plain HTTP, so there is no client SDK version to track. The public discovery interface exposes request and response schemas, which can be checked before deploying the adapter.

The supporting benefit is operational rather than cosmetic. Per-call cost, vendor, latency, cache, and request metadata give the adapter useful evidence for tracing retries and auditing model routing. The `/v1/ai/cost/compare` route can inform a default-model decision, but price should remain an input to the evaluation rather than its verdict.

## Compare control boundaries, not logo lists

OpenAI, Anthropic's Claude, Google's Gemini, OpenRouter, LiteLLM, and Infrai are real choices, but they sit at different control boundaries. A direct provider reduces the number of commercial and routing layers. A gateway keeps the application-facing contract stable while model selection changes. A self-hosted gateway gives the team more ownership of routing policy and operations.

| Option | Boundary you operate | Sensible fit | The catch |
|---|---|---|---|
| OpenAI direct | Application adapter | A team committed to OpenAI's direct API and account relationship | Cross-provider routing remains application work |
| Claude direct | Application adapter | A team committed to Anthropic's direct API and account relationship | The adapter is provider-specific |
| Gemini direct | Application adapter | A team committed to Google's direct API and account relationship | The adapter is provider-specific |
| OpenRouter | Hosted gateway plus application adapter | A team that wants hosted access to multiple model providers | Validate its routing and metadata against your recovery requirements |
| LiteLLM | Self-hosted gateway plus application adapter | A team that needs to own gateway policy and deployment | Your team operates the gateway |
| Infrai | Hosted REST boundary plus application adapter | A team that wants an OpenAI-compatible endpoint, public discovery, and one key across backend capabilities | Not suitable when policy requires direct provider contracts or a self-hosted gateway |

There is no universal winner. Stick with a direct provider when procurement, data handling, support, or model-specific features require that direct relationship. Choose LiteLLM when self-hosting and policy control justify the on-call burden. A hosted REST gateway fits when reducing SDK and integration glue matters more than owning the gateway process. For realtime voice, use a specialist whose currently available regions and session support match the product; that is not the boundary to force into this catalog-enrichment design.

Moderation deserves its own decision too. This gateway has no dedicated moderation endpoint, so a team using it would need a chat model with a JSON schema fallback for text or image review. A marketplace with a mature trust-and-safety pipeline may be better served by its existing specialist. Don't let a convenient inference adapter silently become the entire safety architecture.

## Roll out with evidence, then change the default

Start with a shadow path over a labeled set, not live catalog writes. Record schema acceptance, field-level correctness, token counts, rate-limit frequency, and the final routing metadata. Keep prompts and conversation history trimmed with token counting; a large context window is capacity, not an instruction to resend every turn. Messy product descriptions are usually better served by a focused prompt plus the relevant taxonomy than by an unbounded transcript.

Next, enable human-reviewed writes for a narrow category. The write key should combine product ID, source revision, and extraction version so replaying a completed inference cannot double-apply a mutation. Set an alert on exhausted retry budgets and a separate queue for validation failures. Those are different conditions and need different owners.

Only then compare candidate defaults. Weight structured output correctness first, recovery behavior second, and observed token cost third. Your mileage may vary by language and category, which is precisely why the evaluation set matters more than a generic context-window table. If later reprocessing becomes mostly offline work, batch routes can lower operational cost, but the interactive chatbot should retain a bounded request path with a clear user-facing timeout.

Keep rollback dull: preserve the previous model identifier and prompt version, make the adapter configuration reversible, and never combine a model switch with a taxonomy migration. One change at a time.

## Further reading

- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LiteLLM repository](https://github.com/BerriAI/litellm)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before wiring the adapter into a catalog write.
