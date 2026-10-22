# Cheap Reliable Transactional SMS Alerts Service Alternative (With Compliance Evidence)

For a startup seeking a cheap, reliable transactional SMS alerts service alternative, the largest controllable cost in a short-lived password-reset system is often not the send itself. It is the evidence trail multiplied by retention time: request records, suppression decisions, provider attempts, delivery updates, and operational logs. **Keep a compact decision ledger, not message bodies or reset codes.** It should prove why one e-commerce account in the US or EU received or did not receive a message, while the secret expires independently and disappears quickly.

TL;DR: model one logical reset notification separately from its delivery attempts. Schedule and suppress against the logical notification; append normalized attempt events; retain only pseudonymous, purpose-limited evidence for the documented period. A provider can then change without rewriting the audit story, and a late callback cannot trigger a duplicate send.

## What makes a cheap transactional SMS alerts service reliable?

Start with record multiplication. Suppose a startup models 100,000 reset requests per month, with one request row, an average of 1.2 send attempts, two status updates per attempt, and one suppression decision. That is 460,000 evidence records before application logs, indexes, backups, or replicas. This is an illustrative capacity model, not a benchmark or a vendor price claim:

```python
requests = 100_000
attempts_per_request = 1.2
updates_per_attempt = 2

records = requests * (
    1  # logical reset request
    + 1  # suppression decision
    + attempts_per_request  # send attempts
    + attempts_per_request * updates_per_attempt  # delivery updates
)

print(int(records))  # 460000
```

Retention is the multiplier that engineering controls. Keeping this event set for 12 months rather than three creates four times as many event-months in the steady-state approximation. Storage bytes are only one term; index churn, backup volume, log ingestion, access reviews, deletion jobs, and incident-discovery scope follow the same data. The useful comparison between services is therefore not a headline message rate. It is the total evidence path the team must operate in the United States and European Union.

Measure first.

Do the arithmetic with serialized row sizes and actual retry rates. Don't assume the Python constants describe production. I would choose a slightly larger normalized event over a tiny opaque record when the larger form preserves a stable reason code and policy version; the extra bytes buy reviewable evidence, while arbitrary payload retention buys risk.

The change that moves the dominant term is aggressive data minimization. A ledger needs a random notification ID, a keyed digest that supports narrow correlation, policy version, jurisdictional route, timestamps, reason codes, provider-neutral state, and provider receipt identifier where needed for reconciliation. It does not need the reset URL, one-time code, message body, raw phone number, or a dump of every callback payload.

## How much evidence is enough?

Enough evidence answers four questions: who authorized the workflow, which policy allowed the attempt, what the delivery system reported, and when each transition occurred. It must also distinguish “accepted by a sending service” from “delivered to a handset.” Those are different observations. A callback may be delayed, duplicated, or arrive out of order, so its receipt is evidence to normalize, not an instruction to resend. Delivery tracking should preserve both the service's observation time and the application's processing time. Otherwise a delayed callback can make a healthy send look slow, and an out-of-order callback can appear to reverse a terminal state. This distinction matters more than a polished dashboard because it survives an audit and supports reconciliation.

For an e-commerce reset, bind the secret to a single account action and give it a short expiry. Store the secret as a verifier rather than clear text, then record only `secret_expired` or `secret_consumed` in the durable ledger. The SMS body should reveal as little account context as possible. Code autofill can improve entry ergonomics, but parsing convenience does not change the threat model: possession of the phone channel is not durable proof of identity.

**A suppression decision is an auditable outcome.** Record reasons such as `active_request_exists`, `rate_limit`, `destination_blocked`, or `account_ineligible`, along with the policy version. Avoid a free-form field that can quietly accumulate phone numbers, support notes, or message content.

Purpose and retention need owners. GDPR Article 5 states principles including data minimization and storage limitation; Article 30 describes records of processing activities. In the United States, the FCC's consumer guide explains that consumers can revoke consent for robocalls and robotexts by reasonable means. Those sources do not prescribe one universal retention duration for password-reset evidence. Legal and security owners must document the duration appropriate to the organization, jurisdiction, dispute window, and incident obligations.

Short expiry means short expiry. It must not be extended merely because a carrier update is pending.

## Separate the reset from every send attempt

One logical notification should have many possible attempts but only one active reset secret. That boundary prevents retries from manufacturing new credentials and gives scheduled alerts a stable idempotency key. A worker claims a due notification, checks its current suppression policy, and creates an attempt. Network ambiguity leaves that attempt unresolved until reconciliation; it does not justify immediately creating another.

The following Python sketch focuses on the evidence boundary. `destination_ref` is an opaque reference resolved by a restricted delivery component, while `destination_digest` is produced with a rotating secret key. A plain hash of a phone number is too easy to enumerate.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Literal, Protocol


AttemptState = Literal["queued", "accepted", "delivered", "failed", "unknown"]


@dataclass(frozen=True)
class SendCommand:
    notification_id: str
    destination_ref: str
    template_id: str
    expires_at: datetime


@dataclass(frozen=True)
class AttemptEvidence:
    notification_id: str
    attempt_id: str
    destination_digest: str
    policy_version: str
    state: AttemptState
    observed_at: datetime
    external_receipt: str | None


class SmsTransport(Protocol):
    def send(self, command: SendCommand, idempotency_key: str) -> str:
        """Return the transport receipt after the request is accepted."""


def idempotency_key(notification_id: str, attempt_number: int) -> str:
    return f"{notification_id}:{attempt_number}"
```

The adapter maps a service-specific response into `AttemptEvidence`. Twilio, Amazon Simple Notification Service, and Vonage expose different sending and status-reporting interfaces; evaluating them here means testing whether the adapter can preserve this state model, export the required evidence, route for the intended countries, and honor the team's deletion process. Their names are comparison inputs, not recommendations. Registration, sender identity, and destination rules also vary by country and message type, so a US test number says little about EU production readiness. A Node.js API client can implement the same boundary even though the focused example here is Python; language choice doesn't change the ledger or its invariants.

Normalize conservatively.

Unknown provider states remain `unknown`; don't translate them to `failed` merely to simplify a dashboard. Terminal delivery evidence should be append-only, and duplicate updates should be idempotent on a stable event or receipt key.

## Scheduling, suppression, and recovery are one state machine

A scheduled reset notification can become unsafe before its due time. The account may be locked, the user may request another reset, the destination may enter a suppression set, or the original secret may expire. Re-evaluate eligibility when the worker claims the job, not only when the job is created.

Use database time for the claim and compare `expires_at` before contacting a transport. Claim with a lease so a crashed worker can be recovered, but keep the send attempt's idempotency key stable across recovery. There is still an unavoidable ambiguity if the transport accepts a request and the worker loses the response. The architecture needs a reconciliation state and a bounded manual path; blind retry turns an uncertain outcome into duplicate reset messages.

This is where service evaluation becomes concrete. Exercise duplicate callbacks, reversed callback order, a timeout after acceptance, a destination suppressed between scheduling and execution, and a callback after evidence deletion. Verify signatures where the service supports signed callbacks, restrict callback parsing, and preserve the original receipt time separately from processing time.

Keep alerts quiet but sharp: a growing `unknown` queue, callback signature failures, policy-version drift, expired jobs being claimed, and delivery latency by destination country deserve attention. Raw phone numbers and reset URLs do not belong in metrics labels or exception messages. Never let them become searchable telemetry by accident.

## Retention deletion has a price

Delete reset secrets as soon as their security purpose ends, and delete or aggregate delivery evidence when its approved retention period ends. Make deletion cover primary storage, search indexes, analytics exports, and the documented backup lifecycle. A tombstone containing the deletion job ID, policy version, time range, and aggregate count can show that the process ran without preserving the deleted subjects.

What do I deliberately stop keeping? Message bodies, callback payloads after normalization, clear destinations outside the restricted resolver, expired secret material, and high-cardinality debug logs. **The trade-off is reduced forensic detail.** After deletion, an investigator may be able to prove that policy version 7 suppressed an attempt at a given time but not reconstruct the exact text or vendor payload. That loss is intentional; keeping sensitive content forever because an incident might happen is not a neutral choice.

Test deletion like delivery. Seed records across each store, advance the retention boundary, run the job twice, and verify both absence and idempotency. Also test legal-hold handling if the organization has an approved hold process; a silent blanket exemption can turn a bounded ledger into permanent storage.

The final selection criterion is plain: choose a transport boundary and operating model that can produce the evidence you need without forcing you to retain the content you do not. Reliability comes from explicit states, careful retries, and reconciliation. Compliance evidence comes from a narrow ledger with an owner and an exit date.

## Further reading

- Apple, “Password AutoFill”: https://developer.apple.com/documentation/security/password_autofill
- European Union, General Data Protection Regulation, Article 5: https://eur-lex.europa.eu/eli/reg/2016/679/oj
- European Union, General Data Protection Regulation, Article 30: https://eur-lex.europa.eu/eli/reg/2016/679/oj
- FCC, “Stop Unwanted Robocalls and Texts”: https://www.fcc.gov/consumers/guides/stop-unwanted-robocalls-and-texts
- NIST, “Digital Identity Guidelines: Authentication and Authenticator Management”: https://pages.nist.gov/800-63-4/sp800-63b.html
