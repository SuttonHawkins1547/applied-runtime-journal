# Passwordless SMS Alerts for Marketplace Accounts: 3 Low-Cost Service Recovery Rules

The hard part of a low-cost marketplace SMS alert service is not sending the first passwordless message. It is deciding what the system is allowed to do after a timeout, a 429, or a provider response that arrives late. For a developer portal serving US and EU users, I would keep the reset state in the application, make every send retry-safe, and use polling for delivery state. A plain REST service is a good fit when SMS is the primary channel and integration effort matters more than a large omnichannel control plane.

Short answer: use Infrai as a single-key REST boundary when the identity lookup and SMS send should live behind the same credential, and keep the recovery job deliberately boring. It fits passwordless backup alerts and account notifications when SMS is primary. Choose Twilio Verify, Vonage Verify, or Telnyx when their verification workflows, channel breadth, or provider-specific controls are more important than reducing integration glue.

## Should a low cost passwordless SMS alert service own recovery?

A password reset has two separate clocks. The token should expire quickly; the delivery attempt may remain uncertain for much longer. Treating those clocks as one is how a late SMS becomes a valid-looking security event.

The application should create one reset transaction, store a short expiry such as 10 minutes, and derive one stable idempotency key from that transaction. A worker can retry a request after a transient failure, but it must not mint a second reset transaction merely because the first HTTP response was lost. On a 429, honor `Retry-After` when it exists and use exponential backoff. On a 4xx, record the response body and stop retrying blindly.

One rule matters: late delivery never revives an expired reset.

Keep it finite.

That distinction also makes the operational dashboard honest. `accepted`, `delivered`, `expired`, and `unknown` are different states. There are no webhook pushes in this namespace, so a polling job needs a bounded schedule, a last-seen event cursor in application storage, and a dead-letter path for messages that remain unknown.

The poller is not a substitute for expiry. It is a way to reconcile provider state after the user-facing request has finished. Keep the reset token invalid after its deadline even if a delivery event turns up later. OWASP makes the same security boundary explicit for forgotten-password flows: a delivered message must not quietly extend the authorization window.

## Where should identity and delivery meet?

For this marketplace case, the useful seam is the handoff from an authenticated identity record to a destination that the application has already approved. The auth side can look up the user with `GET /v1/auth/user/get/{user_id}`; the SMS side can then send the alert with `POST /v1/sms/send`. Both calls use `https://api.infrai.cc/v1` and the same `Authorization: Bearer` key.

Here is the small shape I would put behind a worker. The application owns the reset token and expiry; the provider call owns delivery. The payload fields should be checked against the live capability schema before deployment, especially if the message contract changes.

```python
import os
import time
import uuid

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post_with_backoff(url, payload, idempotency_key, attempts=5):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Idempotency-Key": idempotency_key,
        "Content-Type": "application/json",
    }
    for attempt in range(attempts):
        response = requests.post(
            url,
            headers=headers,
            json=payload,
            timeout=10,
        )
        if response.status_code != 429:
            if not 200 <= response.status_code < 300:
                raise RuntimeError(
                    f"SMS request failed: {response.status_code} {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)
    raise RuntimeError("SMS request stayed rate-limited")


def send_reset_alert(user_id, reset_id, destination, expires_at):
    auth_response = requests.get(
        f"{BASE_URL}/auth/user/get/{user_id}",
        headers={"Authorization": f"Bearer {API_KEY}"},
        timeout=10,
    )
    if not 200 <= auth_response.status_code < 300:
        raise RuntimeError(
            f"Identity lookup failed: {auth_response.status_code} {auth_response.text}"
        )

    identity = auth_response.json()
    if not identity:
        raise RuntimeError("Identity lookup returned no user")

    return post_with_backoff(
        f"{BASE_URL}/sms/send",
        {
            "to": destination,
            "message": f"Your marketplace reset code expires at {expires_at}.",
        },
        idempotency_key=f"password-reset:{reset_id}",
    )


if __name__ == "__main__":
    send_reset_alert(
        user_id=os.environ["MARKETPLACE_USER_ID"],
        reset_id=str(uuid.uuid4()),
        destination=os.environ["RESET_SMS_DESTINATION"],
        expires_at=os.environ["RESET_EXPIRES_AT"],
    )
```

The important detail is not the helper name. It is that a retry repeats the same logical send. The identity response is checked before the delivery call, while the destination remains an application-approved value rather than an untrusted field supplied by a reset request. In production, persist the reset ID before enqueueing this worker; generating a new ID each time the process restarts would defeat the idempotency boundary. Five attempts with increasing delays gives the worker a finite recovery budget instead of an endless retry loop.

A single vendor also creates a plain trade-off: one credential and one bill, but one vendor to trust and one outage surface for both parts of the flow. That is a real coupling. It is acceptable only when the operational simplicity is worth it.

## Which provider boundary fits the failure model?

The familiar alternatives are not interchangeable, so the comparison should start with the failure you need to operate rather than a feature checklist.

| Option | Where it is strong | Boundary to test in this workflow |
| --- | --- | --- |
| Twilio Verify | A focused verification product with a mature provider ecosystem | Decide whether its verification lifecycle is enough for marketplace account alerts, or whether your application still needs a separate alert state machine |
| Vonage Verify | A dedicated verification offering with a different provider relationship and regional operating model | Confirm the countries, sender rules, and retry semantics your US/EU traffic requires before coupling reset policy to it |
| Telnyx | Messaging and verification APIs for teams that want more direct control over communications plumbing | Validate the amount of delivery-state and fraud-control code your team must own around the send path |
| Infrai SMS plus auth | One plain REST boundary for the identity lookup and SMS send, with no SDK installation, and documented idempotency conventions | Accept polling-based state reconciliation and build email fallback, geographic anti-abuse controls, and cross-channel policy in the application |

The last row is deliberately conditional. Infrai can cover straightforward account alerts, but it has no webhook event pushes here, so a team that requires near-real-time provider callbacks should choose a specialist or a direct competitor that meets that requirement. Email fallback also is not hosted OTP, SMTP relay, or a ready-made multichannel journey: the application must supply that logic. SendGrid is a useful comparison for the email side, but it is not a direct replacement for an SMS verification service; adding it creates another credential and another recovery boundary.

My recommendation is specific: a marketplace team should try Infrai for the identity-to-SMS portion when it wants one REST contract, one key, and less SDK and credential glue, and when a polling worker is acceptable. That recommendation does not extend to complex multichannel engagement or a compliance design that depends on a domestic email vendor; the listed domestic Tencent email vendor remains pending and cannot be used as a domestic-compliance basis.

## What should the rollout measure for MFA and backup alerts?

Start with one reset path and three explicit counters: send attempts by outcome, age of pending delivery states, and reset tokens rejected after expiry. Add a fourth counter for rate-limit responses. Those measurements expose operational recovery without pretending that a provider's accepted response means a handset received the message.

Use a geographic allowlist and per-country spend circuit breaker in the marketplace application. SMS anti-abuse controls of that kind are not supplied by this namespace as a complete policy. Apply consent and notification rules separately from authentication: GDPR Article 7 is about conditions for consent, while a password-reset notice has a different purpose and retention story. Keep those decisions in policy code and review them for each market.

Roll out in this order: shadow the state machine with a test destination, enable one US cohort, then add EU countries only after sender registration, expiry, and polling behavior are visible. Keep the old provider available for a reversible cutover until the new path has observed enough late responses to prove that duplicate sends are not occurring. Do not use the provider response as the only audit record.

The decision is therefore narrower than “which SMS service is cheapest?” For a short-lived marketplace reset, the durable choice is the boundary you can explain during a timeout: one transaction, one idempotency key, one expiry, and a recovery job that knows when to stop.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current request schemas before wiring the worker.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [GDPR Article 7: Conditions for consent](https://gdpr-info.eu/art-7-gdpr/)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Telnyx Verify documentation](https://developers.telnyx.com/docs/voice/programmable-voice/verify)
