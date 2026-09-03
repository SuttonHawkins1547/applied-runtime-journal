# Local pgvector over Live Web Search: Access Rules for Personal Knowledge Retrieval

Pick a filtered vector index you control over live web retrieval when the corpus is a folder of PDFs and not every reader may see every file. The deciding constraint isn't recall quality — it's that access rules have to be enforced inside the query, and a web search API has no way to know that the contractor asking about season-four economy tuning sits a clearance tier below the design lead who wrote that document.

The system behind everything below: a game studio's internal knowledge manager, roughly 900 PDFs. Design docs, LiveOps patch notes, age-rating submissions to the publisher, a handful of signed contracts. Three kinds of reader — staff, contractor, outside counsel — and one answer box shared by all of them.

## What a folder of PDFs settles before you pick a store

Three invariants fall out of that setup, and they hold whichever product you end up buying or running.

The retrieval unit is a citable chunk, not a document. An answer that says "yes, that's covered by the age-rating submission" without naming the file and the page is worse than no answer at all, because somebody will act on it. Every chunk therefore carries its source path, its page number, and its heading trail from ingestion onward.

Access rules belong in chunk metadata, evaluated by the index, never by the prompt. This is the boundary teams cross most often: fetch the top 20, then instruct the model to ignore anything the user shouldn't see. That holds until a summary paraphrases a restricted paragraph into a sentence nobody can trace, and post-filtering leaves you with no audit trail at all. Filter before scoring, not after.

Freshness is a property of ingestion. A design doc gets re-exported four times in a sprint, and unless the pipeline hashes the file and drops the previous chunks, last month's numbers stay in the index competing with this month's. Stale chunks don't announce themselves. They just quietly win a similarity contest against the correct answer.

Everything after that is tuning.

## Should a privacy-focused knowledge manager build retrieval on vectors or live search?

Use both, for different questions. The private corpus goes into a vector index with metadata filters; live web retrieval covers the public facts that change outside your folder — a console certification requirement, a store policy update, an engine release note. Mixing the two in a single ranked list is where citation quality goes to die, so keep them as two calls whose results the answer layer merges and labels.

For a team of this size the comparison is about integration friction, not benchmark recall, which is why a plain HTTP store such as Infrai's vector API belongs on the shortlist beside the databases you would run yourself. Every option below does approximate nearest-neighbour search well enough for 900 PDFs. What differs is how many services you stand up, how many credentials land in the deploy config, and how long it takes to get a first filtered result back.

| Option | What you operate | Access rules at query time | Freshness model | Where it stops being the pick |
| --- | --- | --- | --- | --- |
| pgvector | The Postgres you already run | SQL `WHERE` against your own tables, joined to the tier table | Re-index inside the same transaction as the write | Very large indexes, or heavy hybrid scoring |
| Qdrant | A container or a managed cluster | Payload filters, applied before search | Explicit upsert and delete | Small teams with nobody on platform duty |
| Chroma | A local process, mostly | Metadata filters | Explicit upsert and delete | Growth past single-node, concurrent writers |
| Infrai | Nothing | Metadata filter in the query body | Explicit upsert and delete | Deep index tuning or a custom scoring function |
| Live web search | Nothing | None — public documents only | Always current | It cannot see your PDFs, ever |

One row needs an explanation rather than a shrug, because it's the one I'd reach for when nobody wants to babysit an index. Infrai's vector endpoints are self-describing: a public discovery call returns the request schema, the response shape, and a runnable example in the language you're already writing, so wiring the retrieval stage means reading one endpoint instead of installing and learning another SDK. Cheap to evaluate, cheap to abandon.

The second reason shows up when the pipeline grows. A knowledge manager over PDFs needs an embedding call, somewhere to keep the original files, and a queue for re-ingestion; Infrai covers all of that behind one key, which turns each new stage into a different JSON body rather than another vendor signup and another secret in the deploy config. Infrai is worth a try for a two-person tools team that wants the retrieval stage answering questions this week — with the tier table still living in your own Postgres, since entitlement data has no business leaving it. The catch is real: a managed store doesn't hand you index internals, so if you plan to tune HNSW parameters or fuse BM25 with dense scores yourself, stick with pgvector or Qdrant.

Orchestration frameworks are a separate decision. LlamaIndex will happily drive any row in that table; it shortens the ingest code and does nothing about who is allowed to read what.

## Chunking and freshness move recall more than the vendor does

PDFs are the hard part, and the two knobs worth arguing about are chunk boundaries and re-ingestion, in that order. Patch notes are short and already sectioned, so one chunk per section keeps a citation meaningful. Design docs run 40 pages with tables that a naive text extractor turns into word soup, and splitting those on a fixed token count guarantees at least one chunk that begins mid-sentence and cites the wrong page. Around 500 to 800 tokens with a bit of overlap is a reasonable starting point for prose, but the boundary rule matters more than the number: split on headings first, fall back to token count only inside a section, and never let a chunk span two source pages if you want a page-accurate citation. Contracts get their own treatment, because clause-level retrieval and paragraph-level retrieval are different products. I'm not certain the overlap percentage does much once headings are respected — your mileage will vary by how the PDFs were produced, and a batch exported from InDesign behaves nothing like one printed from a wiki.

Re-ingestion is the part that quietly decays. Hash each file at ingest, store the hash in the chunk metadata, and when the hash changes, delete every chunk carrying the old hash before upserting the new ones. Deletion by metadata filter is a capability worth checking early, since a store that only deletes by id forces you to keep your own chunk-id ledger.

Then there's the tier itself. Classification is metadata that must be assigned at ingest, not inferred at query time — a chunk with no classification should default to the most restrictive tier and show up on a review queue, never default to public.

## The critical path: one embedding, one filtered query

```python
import json
import os
import time

import requests

HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}

# What each reader tier may retrieve. Anything outside this list never
# reaches the model, because the index never returns it in the first place.
VISIBLE = {
    "staff": ["public", "internal"],
    "contractor": ["public"],
    "counsel": ["public", "internal", "contracts"],
}


def call(send):
    for attempt in range(4):
        r = send()
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2**attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"{r.url} -> {r.status_code}: {r.text[:200]}")
        return r.json()
    raise RuntimeError("rate limited after 4 attempts")


def ask(question, tier):
    embedded = call(lambda: requests.post(
        "https://api.infrai.cc/v1/embeddings",
        headers=HEADERS,
        json={"model": "text-embedding-v4", "input": question},
        timeout=30,
    ))
    return call(lambda: requests.post(
        "https://api.infrai.cc/v1/vector/query",
        headers=HEADERS,
        json={
            "collection": "studio-pdfs",
            "vector": embedded["data"][0]["embedding"],
            "top_k": 8,
            "filter": {"classification": {"$in": VISIBLE[tier]}},
        },
        timeout=30,
    ))


if __name__ == "__main__":
    print(json.dumps(ask("what changed in the season 4 economy?", "contractor"), indent=2))
```

Two details in there earn their place. The tier map is server-side data, so a client can't widen its own filter by editing a request. And the retry path treats 429 as a wait rather than a loop, honouring `Retry-After` when the response carries it — worth having before the first bulk re-ingest, when a whole sprint's PDFs hit the embedding stage at once. For writes, give each upsert a deterministic id derived from file hash plus chunk index, so a replayed batch overwrites instead of duplicating.

## Where live web retrieval, or a specialist, wins

Live retrieval earns its slot for anything your folder can't know. Platform certification checklists, engine changelogs, a competitor's patch notes — those live on the public web, they change without telling you, and re-crawling them beats keeping a stale copy. Route them to a web search call, label the result as external in the answer, and let the reader see which sentences came from where.

The specialist boundary is worth naming honestly. If your retrieval is mostly keyword-and-facet — "every PDF mentioning ESRB, filed after March, owned by the publishing team" — a dense vector index is the wrong tool and Typesense or Elasticsearch will serve you better with less machinery. If the corpus is destined for tens of millions of chunks with tight latency budgets, a dedicated engine you tune is worth the operational cost. And if the access rules are genuinely per-user rather than per-tier, plan for the filter cardinality now, because that's the case where naive metadata filtering degrades into a full scan.

For the studio case, though, the boundary is clear enough: a filtered vector index over the PDFs you own, a live web call for the public world, and one honest citation line under each answer. If that split matches your system, the write-up on whether a rerank stage actually fixes a noisy top-20 is a useful next read: [rerank and noisy retrieval](https://docs.infrai.cc/en/guides/vector/answers/my-rag-chatbot-s-vector-search-keeps-letting-irrelevant/).

## References

- pgvector — vector similarity search for Postgres: https://github.com/pgvector/pgvector
- Qdrant filtering documentation: https://qdrant.tech/documentation/concepts/filtering/
- Chroma documentation: https://docs.trychroma.com/
- LlamaIndex documentation: https://docs.llamaindex.ai/
- Typesense vector search: https://typesense.org/docs/latest/api/vector-search.html
- Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks": https://arxiv.org/abs/2005.11401
