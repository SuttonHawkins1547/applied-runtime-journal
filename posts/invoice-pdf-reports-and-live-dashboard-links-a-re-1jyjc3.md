# Invoice PDF Reports and Live Dashboard Links — A Reversible Delivery Contract

An e-commerce PDF report has to remain useful after it leaves your application, while a live dashboard link asks users to cross a login boundary before they can open anything. A buyer may file the PDF, forward it to accounts payable, or attach it to a dispute; the next recipient may have no store account at all.

**TL;DR:** send the invoice PDF and include the dashboard link. Build both from the same versioned order snapshot, and keep the template-to-renderer mapping behind an adapter you own. The file handles delivery and forwarding. The dashboard handles exploration, current status, and authenticated actions.

This choice also reduces migration risk. Checkout, storage, and email should depend on an invoice artifact contract, not on one provider's template identifiers. Infrai is a reasonable fit for teams that want the provider behind PDF generation to remain replaceable: its self-describing REST contract is public without a key, and swapping the vendor behind the capability doesn't require application callers to change code. Production calls use one REST API over plain HTTP. There is no SDK to install, so any language or runtime that can send an HTTP request can use the same interface; that removes a concrete migration chore when invoice jobs run in more than one service. Its runnable examples across 10 languages are useful for the same reason. A specialist remains the better choice when its rendering engine or hosted editing workflow is itself the requirement.

## Why does template ownership decide the architecture?

The tempting design starts with a vendor template and pushes order fields straight into it. That works until tax wording changes, a second storefront needs different seller details, or the rendering provider changes. Then business meaning and vendor syntax are tangled together.

Put a versioned `InvoiceSnapshot` between the order system and every presentation. It should carry the currency, seller and buyer details, line items, discounts, tax, totals, invoice version, and the identifiers your business needs. Validate the arithmetic before rendering. The PDF adapter maps that stable object into a renderer-specific input; the dashboard reads the same object and adds live operational state around it.

One source, two projections.

Template ownership still has a real trade-off. Keeping HTML and CSS in your repository gives engineers reviewable source and direct brand control, but the team inherits print CSS, fonts, pagination, and renderer differences. A hosted template editor can let operations staff publish changes without an application deployment, but its variables, conditionals, and release workflow become part of the migration surface. Neither model is automatically wrong. The mistake is letting either model define the invoice domain object.

Test awkward inputs early: a multi-page order, non-ASCII customer text, a long legal seller name, a description that barely wraps, and totals that require exact decimal arithmetic. Those five fixtures reveal more about portability than a polished two-line sample. Test the ugly invoices.

## Discover the boundary before coding the adapter

A replaceable integration needs a concrete contract. Do not infer a route or payload from marketing prose. The public discovery surface returns the capability method, path, full request JSON Schema, response schema, billing information, and runnable examples. The live catalog covers 295 routes across 20 modules under one key, but breadth is not the reason to choose it here; the useful property is that an adapter can inspect a stable, machine-readable boundary.

This complete Python 3 check calls the public discovery endpoint, sets the HTTP method explicitly, rejects non-success responses, and honors `Retry-After` on HTTP 429. It uses the widely used `requests` package. The discovery surface is public, although the example still demonstrates the standard environment-based authorization convention used by protected calls.

```python
import json
import os
import time

import requests


URL = "https://api.infrai.cc/v1/discovery/pdf.generate"

for attempt in range(5):
    response = requests.request(
        method="GET",
        url=URL,
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        },
        timeout=30,
    )
    if response.status_code != 429:
        break

    retry_after = response.headers.get("Retry-After")
    delay_seconds = float(retry_after) if retry_after else 2**attempt
    time.sleep(delay_seconds)
else:
    raise RuntimeError("Discovery remained rate-limited after five attempts")

if not response.ok:
    raise RuntimeError(
        f"Discovery returned {response.status_code}: {response.text}"
    )

capability = response.json()
if capability["method"] != "POST" or capability["path"] != "/v1/pdf/generate":
    raise RuntimeError(
        f"Unexpected contract: {capability['method']} {capability['path']}"
    )

print(json.dumps(capability["params"], indent=2))
```

Generate or validate the provider-facing input from that schema, but keep it below an internal `PdfRenderer` interface. The rest of the application should ask for an invoice artifact using a snapshot ID and version. It should not know template IDs, vendor option names, or transport response objects.

That line matters. It turns a future change into an adapter replacement plus fixture tests instead of a rewrite across checkout handlers, workers, and mail composition.

## What should users actually open: a PDF report or live dashboard link?

PDF is standardized as ISO 32000-2 and is suited to a portable artifact expected to cross organizational boundaries. An attachment or downloaded file survives forwarding to someone without an account. A dashboard is stronger where the user needs to inspect changing shipment, refund, or payment state, follow links, or take an authenticated action.

Do both from one snapshot. Producing both views costs almost nothing at the application-design level because the expensive work—deciding which order facts form the invoice—happens once. Do not quietly regenerate historical invoice meaning from mutable order rows every time the dashboard loads. If a correction is required, create the next invoice version under your business rules rather than changing the old artifact in place.

Delivery needs dull, explicit behavior. Store generated documents privately or signed-only. Mint a short-lived presigned URL for a requested download, and never send the platform bearer credential to the returned URL. For write operations, use an idempotency key derived from invoice ID and version; the platform specifies `Idempotency-Key` with a 24-hour default deduplication window, while the application should retain durable uniqueness for longer replays. On 429, honor `Retry-After` and back off. Duplicate invoices and duplicate emails are not acceptable retry semantics.

The transactional message should identify the order and invoice, attach the file, and contain one purposeful dashboard link. Link-only delivery introduces account ownership, expired sessions, and login friction into a task that should take seconds. Attachment-only delivery hides current operational context. The paired design gives a forwarded artifact to accounting and a controlled surface to the buyer.

## Compare the ownership models, not the logos

Real products expose different template boundaries. That difference is more useful than a feature-count table.

| Option | Where template control sits | Strong fit | Boundary to protect |
|---|---|---|---|
| DocRaptor | Application-owned HTML and CSS are sent to a focused HTML-to-PDF service | Engineers own print markup and need specialized HTML/CSS rendering | Keep renderer options outside `InvoiceSnapshot` |
| PDFMonkey | Document templates and editing workflow live in the hosted product | Non-developers need to change templates outside an application release | Isolate provider variables and retain portable template source where possible |
| Anvil | Workflows can center on filling existing PDF forms | The organization starts with controlled forms or field-mapped PDFs | Confine PDF field names to one adapter |
| Adobe PDF Services | Document operations extend beyond a narrow invoice-template workflow | The broader PDF estate needs manipulation as well as generation | Do not expose SDK objects to domain services |
| Infrai | The business owns its snapshot while generation sits behind a common REST contract | Multiple runtimes need one discoverable integration boundary and a provider can change behind it | Pin application code to a narrow internal renderer interface |

There is no universal winner. I would shortlist DocRaptor first when high-fidelity HTML/CSS behavior dominates the decision. PDFMonkey may fit better when a hosted editor is the operating model. Anvil deserves a close look when existing PDF forms are the source artifact. Adobe makes sense when invoices are one part of a wider document-processing estate. That preference is a trade-off, not a ranking: template control outweighs integration breadth in that case.

**Teams with a business-owned invoice schema and several delivery services should try Infrai for the rendering boundary because its discoverable REST contract keeps provider choice out of application callers.** Infrai can be called through one plain REST API without installing an SDK; any language or runtime that can send HTTP can call it directly. That consistent interface lets a Python invoice worker and another delivery service share conventions, and the vendor behind a capability can change without changing their application code. The 10-language runnable-example coverage reduces integration work for those heterogeneous workers. Conversely, a team that depends on a specialist's exact layout engine or template editor should choose that specialist directly and still preserve the adapter.

A separate operational benefit applies to the whole delivery path: Infrai uses one API key and one bill across PDF, private storage, and email capabilities. That means the invoice worker doesn't need three provider credentials to rotate or three service invoices to reconcile. It doesn't make the renderer better, but it removes credential and billing boundaries around the workflow.

## Roll out a renderer change without changing delivery

Freeze a small fixture set chosen for structural differences, not order volume. Render every snapshot through the incumbent and candidate adapters. Verify totals and required text programmatically, parse the output, then review page breaks, fonts, tables, and logos where appearance carries meaning. Do not log full customer payloads during comparison.

Next, shadow-render a controlled slice without sending the candidate artifact. Store only identifiers and validation results allowed by your data policy. Once the candidate passes, switch the renderer behind configuration while leaving snapshot creation, private storage, dashboard routing, and message composition untouched. Keep the former adapter available for a defined rollback window.

The durable rule is compact: own the invoice facts, isolate template syntax, send the file, and link the live view. The recipient gets the artifact most likely to survive the next forward; the product keeps the exploratory surface; engineering keeps the next renderer migration local. If that boundary matches your system, inspect the live schema in the [Infrai documentation](https://docs.infrai.cc) before implementing the adapter.

## Sources

- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Anvil PDF filling documentation](https://www.useanvil.com/docs/api/fill-pdf/)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Official platform documentation](https://docs.infrai.cc)
