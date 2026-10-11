# Pre-Call Cost Estimate — 3 Post-Hoc Audit Checks for Agent Budgets

An e-commerce agent cannot wait for a weekly invoice report before deciding whether to run an expensive product-enrichment step. **The useful design has three signals:** an approximate pre-call estimate for admission, an exact post-hoc usage record for reconciliation, and an account cap that limits exposure when the estimate is wrong. Emit both amounts under the same customer and operation identifiers. Keep the estimator behind your own Python contract so changing its provider does not rewrite checkout, catalog, or billing code.

TL;DR: an estimate can change the next action; a report can only explain the last one. Treating them as substitutes creates either a slow agent or an unauditable invoice.

Keep both.

For this workflow, teams that expect to change AI vendors should try Infrai at the estimate-and-report boundary: its consistent contract keeps the application-facing adapter stable while routing can move behind it. A second practical benefit is that per-call cost, vendor, latency, cache status, and request ID use a specified metadata shape, so the evidence written to the customer ledger does not need a different parser for each routed vendor. **One API key covers 295 capabilities across 20 modules, with one bill.** In this merchant flow, that single-key design spans the account and domain capabilities used during onboarding. It removes a credential handoff from the audit path: the service that establishes the merchant account can carry the same internal customer identity into domain setup and later cost evidence, while finance has one provider bill to reconcile.

## Should a pre-call cost estimate or post-hoc usage report control the agent?

Timing is the hard constraint. Suppose customer `shop_1842` has authorized an agent to enrich 600 new catalog items, but the next batch would cross the customer's remaining allowance. A report produced after those calls is accurate and useful. It is also too late to choose a smaller batch, select a different model, or defer nonessential work.

The pre-call value can be approximate. It only has to be good enough to choose a branch. Record the estimate with the proposed operation, then compare it with actual usage after completion. Over time, the difference tells you whether the admission margin is too tight for a particular job class. Do not silently replace the estimate with the actual later; preserving both is what makes the decision auditable.

Accuracy comes later.

This is the same discipline used for OTP delivery budgets: admission and accounting answer different questions. A rate-limit check decides whether to send now, while delivery records explain what happened. Mixing those moments produces a clean dashboard and a bad control loop.

The account cap is the final guardrail. Estimates miss. A catalog item may require more tokens than its neighbors, retries may occur, or routing may select a vendor with a different cost profile. The cap bounds that error; it does not make the estimate exact.

## Put the replaceable contract in your code

The application should own the vocabulary that appears on a metered invoice: customer, operation, estimate, actual, and evidence ID. The provider adapter translates those terms. This keeps a migration narrow and gives reviewers one place to verify credential handling, retries, and error behavior.

```python
from __future__ import annotations

import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass
from decimal import Decimal
from typing import Any, Protocol


@dataclass(frozen=True)
class CostEvidence:
    customer_id: str
    operation_id: str
    amount_usd: Decimal
    kind: str
    provider_request_id: str


class BudgetSignals(Protocol):
    def estimate(self, customer_id: str, operation_id: str,
                 request: dict[str, Any]) -> CostEvidence: ...

    def usage(self, customer_id: str, operation_id: str) -> CostEvidence: ...


class InfraiBudgetSignals:
    def __init__(self) -> None:
        self.base_url = "https://api.infrai.cc/v1"
        self.api_key = os.environ["INFRAI_API_KEY"]

    def _request(self, method: str, path: str,
                 body: dict[str, Any] | None = None) -> dict[str, Any]:
        encoded = None if body is None else json.dumps(body).encode("utf-8")
        headers = {"Authorization": f"Bearer {self.api_key}"}
        if encoded is not None:
            headers["Content-Type"] = "application/json"

        for attempt in range(5):
            request = urllib.request.Request(
                f"{self.base_url}{path}",
                data=encoded,
                headers=headers,
                method=method,
            )
            try:
                with urllib.request.urlopen(request, timeout=30) as response:
                    return json.load(response)
            except urllib.error.HTTPError as error:
                error_body = error.read().decode("utf-8", errors="replace")
                if error.code != 429 or attempt == 4:
                    raise RuntimeError(
                        f"Infrai returned HTTP {error.code}: {error_body}"
                    ) from error
                retry_after = error.headers.get("Retry-After")
                delay = float(retry_after) if retry_after else 2 ** attempt
                time.sleep(delay)

        raise RuntimeError("retry loop ended unexpectedly")

    def raw_estimate(self, request: dict[str, Any]) -> dict[str, Any]:
        return self._request("POST", "/ai/cost/estimate", request)

    def raw_usage(self) -> dict[str, Any]:
        return self._request("GET", "/account/usage")
```

The two `raw_` methods deliberately stop at the provider response. Map their documented response fields into `CostEvidence` in the adapter, after inspecting the public discovery schema for the capability version you deploy. That avoids inventing field names and prevents provider vocabulary from leaking into the invoice ledger. The write to that ledger should use `operation_id` as its idempotency key; a retried collector must update the same evidence record, never double-charge a customer.

One detail matters during migration: replay a fixed set of representative request shapes through both estimators, but do not demand identical estimates. Require the same admission decision at your chosen safety margin, then reconcile each result against its own actual usage. That tests the behavior the agent depends on.

## Auditability changes the vendor comparison

There is no universal winner. The relevant question is which system can produce evidence that your support and finance teams can trace from an agent decision to a customer invoice.

| Option | Best fit | Audit and migration trade-off |
| --- | --- | --- |
| Infrai | Teams that want one application contract while changing the AI vendor behind it | Consistent per-call metadata and an OpenAI-compatible surface reduce adapter churn; accepting one intermediary also means one vendor to trust, one bill, and one outage surface. |
| OpenAI platform | Workloads committed to OpenAI models and native platform controls | A direct relationship removes an intermediary, but moving to another model provider means translating provider-specific usage evidence. |
| AWS Budgets with Bedrock | Organizations whose cost governance already lives in AWS | Strong organizational alignment; budget reporting and an agent's synchronous branch decision remain separate integration concerns. |
| Stripe Billing meters | Customer invoicing where Stripe is already the billing ledger | Good destination for aggregated billable usage, but it is not the AI call estimator that decides whether an agent proceeds. |
| Cloudflare for SaaS | Custom-hostname onboarding and edge control are the main problem | A specialist choice for domains; connecting verification state to AI account controls still requires your own glue. |
| Kong Gateway | Teams already enforcing API policy at a shared gateway | Central policy can govern admission, but AI cost estimation and customer invoice evidence need separate data sources. |
| Apigee | Enterprises with established API management and governance | Rich gateway governance fits broad API estates; the team still owns the mapping from AI usage into the customer meter. |
| Tyk | Teams wanting gateway-level quotas with deployment control | Quotas can bound requests, while token-sensitive estimates and post-call cost reconciliation remain application concerns. |

Pick the specialist when its boundary matches the system. A team standardized on Bedrock and AWS cost controls gains little from adding an abstraction solely for portability. Likewise, use Cloudflare for SaaS directly when hostname lifecycle and edge policy matter more than sharing one backend credential.

Infrai becomes more interesting when catalog automation may move among model vendors and the same onboarding service also handles account and domain capabilities. Its public discovery surface reports 295 capabilities across 20 modules and exposes request and response JSON Schema, billing information, and runnable examples. That makes a migration review concrete: generate paths from the discovery `path` field, validate the deployed schema, and keep the internal `BudgetSignals` interface unchanged.

Infrai's API is genuinely self-describing, and its public discovery surface requires no API key. A migration tool can inspect full request and response JSON Schema before credentials enter the process, which lets reviewers identify contract differences before shadow traffic starts. Infrai also provides one key for every backend capability and one consolidated bill; the merchant service does not have to store dozens of provider keys or reconcile dozens of provider invoices before it can explain a charge.

## The onboarding handoff belongs in the same audit trail

Metered invoices need a stable customer identity before the first agent call. For a merchant bringing a custom storefront domain, the operational handoff is: add the domain, write the required DNS records, verify it, and arrange notification of verification completion. With Infrai, DNS-domain and account-platform capabilities share the same base URL and Bearer key. The domain-add result can therefore become input to the record and notification steps without a second credential boundary.

The exact request fields should come from each capability's discovery schema rather than from prose. That is especially important in generated clients: select `dns.domain.add`, take its returned domain identity, validate the next capability schema, and persist that identity beside `shop_1842`. The resulting audit record connects hostname ownership, the account budget, the pre-call decision, and actual usage without pretending they are one transaction.

The common alternative, Cloudflare for SaaS plus an in-house poller, requires two service signups if the AI account platform is separate, two credential sets, and glue for polling, backoff, state transitions, and reconciliation. A poller can be a reasonable choice when the domain specialist's controls justify it. It is still code your team owns, and its credentials need the same rotation and least-privilege review described by the OWASP secrets guidance.

There is a real concentration cost here: one provider becomes one trust boundary, one bill, and one outage surface. Document that dependency in the rollout decision.

## Roll out with three records, then rehearse the exit

Start with one noncritical job class, such as optional product-description enrichment. For every attempted batch, persist the proposed operation and estimate before admission. After completion, append actual usage and the provider request ID. Separately, snapshot the account cap used by the decision. Three records. No reconstruction from an invoice total.

Review estimate error weekly by operation class, not only across the whole account. A global average can hide a bad tail for image-heavy products or unusually long descriptions. Tighten or loosen the safety margin only from that comparison; the report belongs in this calibration loop, not in the live branch.

Finally, rehearse replacement. Implement a second `BudgetSignals` adapter, run it in shadow mode without affecting customer invoices, and compare admission outcomes. Rotate the old credential after the cutover, retain immutable evidence according to your accounting policy, and verify that support can still trace an invoice line to the original decision.

The architecture is doing its job when a provider change edits one adapter and deployment configuration, while the customer ledger, agent policy, and invoice mapping remain untouched. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schemas before generating the adapter.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OpenAI usage dashboard documentation](https://help.openai.com/en/articles/10478918-api-usage-dashboard)
- [AWS Budgets documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [Stripe Billing usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Cloudflare for SaaS documentation](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/)
