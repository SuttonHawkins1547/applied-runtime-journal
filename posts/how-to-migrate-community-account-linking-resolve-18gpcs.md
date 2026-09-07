# How to Migrate Community Account Linking: Resolve Identities Without Accidental Merges

Short answer: keep identity resolution separate from account linking, require an unambiguous match before attaching an external identity, and treat every unlink as a session-continuity decision. During a migration off a managed provider, that boundary matters more than the brand of the replacement.

In a content community, a member can arrive with a Google identity, an email login, and a phone number over several years. The expensive part of moving providers is not the API call. It is preserving the graph of identities without silently joining two people who happen to share a display name or an old email address.

## Start with the cost of keeping identities

The bill in an identity migration is mostly a retention bill: how many identity records, session records, audit events, and recovery contacts you keep while two systems overlap. A short dual-run window reduces login disruption, but every extra day means another copy of mapping data to reconcile and protect. Dropping that data too early creates a different cost: support tickets, locked-out contributors, and a manual account-recovery queue.

I model the dominant term before comparing vendors. If a community has 2 million members and an average of 1.4 linked identities, the migration has to preserve roughly 2.8 million edges, not 2 million users. That is the unit that drives export size, reconciliation work, and the risk surface. The number is an example for planning, not a benchmark.

What I deliberately stop keeping is the old provider's opaque merge decision. I retain the external subject identifier, issuer, link status, and evidence for the decision, then let the new resolver make a fresh, explicit match. When a dispute appears later, the evidence explains why two records stayed separate. That is cheaper than trying to unpick an accidental merge after posts and moderation history have crossed accounts. In a real migration, this means exporting the old links into a staging table, checking issuer and subject uniqueness, replaying only exact matches, and measuring the review queue before the cutover. A member who used a masked relay address may have the same visible email prefix as another member; the prefix is not evidence, and an operator should be able to see that distinction in the audit record.

Do not merge on a hunch.

## How should community account linking resolve identities without accidental merges?

Use a four-state decision: `new`, `matched`, `needs_review`, or `rejected`. First resolve or read the external identity. Only then decide whether it belongs to an existing site user. An exact issuer-plus-subject match can be attached; a fuzzy name, avatar, or reused email cannot.

The safety rule is simple: one external identity may bind to one site user, while one site user may own several identities. Before creating a link, enforce a unique constraint on `(issuer, subject)`. Before removing one, check that the user still has another usable login method. If neither condition is true, pause for an authenticated recovery flow instead of guessing.

Here is the shape I use for a migration worker. The payload comes from the provider's verified token parser, so the example does not invent claim names. The worker sends it to the resolver, retries a rate limit with `Retry-After`, and leaves an ambiguous result for review rather than merging it.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone

import requests


BASE_URL = os.environ["INFRAI_BASE_URL"]


def retry_delay(response, attempt):
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                target = parsedate_to_datetime(value)
                if target.tzinfo is None:
                    target = target.replace(tzinfo=timezone.utc)
                return max(0.0, (target - datetime.now(timezone.utc)).total_seconds())
            except (TypeError, ValueError, OverflowError):
                pass
    return min(30.0, 2 ** attempt)


def resolve_identity(payload):
    key = os.environ["INFRAI_API_KEY"]
    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}/auth/identity/resolve",
            headers={"Authorization": f"Bearer {key}", "Content-Type": "application/json"},
            json=payload,
            timeout=15,
        )
        if response.status_code == 429:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"identity resolution failed ({response.status_code}): {response.text}")
        result = response.json()
        if result.get("status") in {"new", "matched"}:
            return result
        return {"status": "needs_review", "result": result}
    raise TimeoutError("rate limit persisted after five attempts")


payload = json.loads(os.environ["IDENTITY_PAYLOAD_JSON"])
decision = resolve_identity(payload)
print(json.dumps(decision, sort_keys=True))
```

The important judgment is outside the HTTP call. `matched` is not permission to merge arbitrary profiles; it is permission to link only the exact identity the resolver verified. Store the decision and its evidence, then make the link in a transaction guarded by the unique constraint. A retry of this read-style resolution does not create a second identity.

## Compare the migration options by control surface

The managed providers below all support production authentication, but they expose different amounts of control over identity storage and migration timing. Verify current export and retention details against their documentation before signing a plan.

| Option | Identity-linking fit | Migration trade-off |
| --- | --- | --- |
| Auth0 | Mature social connections and account-linking workflows | Broad features, but provider-specific rules and tenant configuration add migration inventory |
| Clerk | Fast community-facing sign-in and user profiles | Pleasant product surface; custom evidence retention and unusual identity graphs may need extra data work |
| Firebase Authentication | Strong fit when the rest of the stack is already Firebase | Portable tokens are useful, yet moving identity data and provider configuration out later takes planning |
| A capability API such as Infrai | One REST contract can keep the resolver call stable while the service behind it changes; one key can cover several backend capabilities | You still own the match policy, uniqueness constraint, audit trail, and recovery UX |

For a small team, the last row can reduce integration surface: Infrai offers one key and one bill for several backend capabilities through a plain REST API, so any language can call it without installing an SDK; swapping the vendor behind a capability does not require rewriting application code. That is a useful property during a provider migration, but it does not turn an ambiguous identity into a safe match.

## Make unlinking a continuity check, not a delete button

An unlink request should load the user's current identities first. The list route is enough to build a preflight: count usable methods, identify the session that initiated the request, and require a recent authenticated step for a high-risk change. If the target is the only usable method, return a recovery challenge instead of removing it.

I initially treated unlink as cleanup. Then I saw how quickly a community member can lose access when a social provider account is closed. The record is tiny; the consequence is not. Keep a reversible audit entry and revoke sessions only according to your incident policy, especially when the reason is a suspected stolen session rather than routine housekeeping.

The catch is that this pattern is not suitable when your product needs provider-managed profile UX, turnkey fraud signals, or a large support operation that cannot own recovery decisions. Stick with Auth0, Clerk, or Firebase when their hosted workflows are the product requirement. Choose a more direct capability layer when the boundary, evidence, and portability are the requirements you need to control.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://clerk.com/docs/guides/development/managing-users
- https://firebase.google.com/docs/auth
- https://datatracker.ietf.org/doc/html/rfc7519
