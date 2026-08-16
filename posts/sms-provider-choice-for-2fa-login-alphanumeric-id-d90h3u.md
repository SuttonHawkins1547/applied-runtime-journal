# SMS Provider Choice for 2FA Login: Alphanumeric Identity and Compliance

Sender approval has to be treated as release state, because an easy API integration cannot compensate for an origination identity that is not cleared for the destination. **Bottom line:** choose a 2FA login SMS provider only after it proves US and EU sender registration, alphanumeric sender handling, local compliance ownership, delivery-state visibility, and support escalation on the routes you will actually use. Keep code generation, retry policy, and recovery outside the transport adapter.

This is an architecture decision record, not a vendor ranking. The decision is to use a narrow internal messaging interface backed by destination-specific configuration and an approved transport. Procurement can change the transport later; authentication semantics stay with the application. Email can be a recovery channel, but it has its own sender requirements and must be evaluated as a separate path rather than assumed to be an automatic substitute.

## What should a 2FA login SMS provider prove about sender registration?

Start with evidence, not the dashboard. For each launch destination, ask a candidate to show which sender identity the proposed route will use, who completes its registration, what evidence marks that registration ready, how a status change is reported, and where an escalation goes. Ask the same questions for an alphanumeric sender and a numeric sender. The answer may differ by destination, so “supports the US and EU” is too coarse to approve a release.

The application team should turn those answers into a route matrix. One row represents a destination and traffic class. Its fields include the requested identity, the approved identity, registration state, message template owner, reply expectation, test numbers, support path, and last verification date. Compliance or counsel supplies the local interpretation; an API response does not. I'm not sure a generic procurement checklist can resolve every local rule, because the missing input is the exact country, route, identity, audience, and message content. A documented review of those inputs can.

Make the launch gate blunt. If the approved identity, consent record, user-support procedure, or controlled delivery test is missing, stop the launch.

No guessing.

Accepted isn't verified.

It describes a handoff at one boundary, not a completed login. Record the provider message identifier and normalize later delivery states, but measure the user outcome separately: a currently valid challenge was verified, rejected, expired, or moved to recovery. This distinction catches a quiet failure mode in which transport charts look healthy while users repeatedly request another code.

The selection packet should therefore contain artifacts that engineering can inspect: the route matrix, registration evidence, a sample status lifecycle, a data-handling review, an account-access model, a support escalation exercise, and a controlled test plan. A polished quickstart is useful during implementation. It isn't approval evidence.

## Invariants and failure boundaries

The authentication service owns the challenge. It generates the code, stores a derived representation rather than plaintext, chooses expiration from security policy, limits verification attempts, applies abuse controls, and consumes a successful challenge once. It also owns resend eligibility. The transport receives an already authorized dispatch command and returns a message identifier; it does not decide whether the user deserves another code. One active code generation should map to one stable dispatch key. A browser refresh, a worker retry, and a second tap must converge on that same operation instead of creating three sends. The durable reservation has to happen before the network call, and competing workers need an atomic result. If the application cannot distinguish “already reserved” from “newly reserved,” it can't reason cleanly about retries. Callbacks cross a different trust boundary. Authenticate them, preserve the raw provider identifier outside user-visible logs, and make state application monotonic according to your documented mapping. A duplicate callback becomes a no-op. An out-of-order callback cannot reopen an expired challenge or consume one. Unknown states go to an operations queue rather than being silently translated into success. Keep secrets out of telemetry — especially the OTP, message body, and full destination. A correlation ID can join the login attempt, dispatch record, and delivery event without turning the log platform into an authentication database. Access to registration records and messaging credentials deserves the same deliberate ownership: named operators, scoped permissions, and a review path when people change roles.

Failure has layers. Registration can be incomplete before deployment. Dispatch can be declined at the application boundary because the retry budget is exhausted. A transport request can be accepted while delivery remains unresolved. A handset can receive a code after the challenge expires. The user can enter a valid code for an older generation. Each layer needs its own state and alert; collapsing them into `sent = true` destroys the evidence needed to debug login gaps.

This is where deliverability and security pull in different directions. Longer validity can tolerate slow arrival, yet it also extends the period in which a code can be used. Aggressive resend may help one delayed user while multiplying messages and confusing another user who receives them out of order. There is no universal timer hiding in a provider feature list. Set the policy from the threat model and observed destination behavior, test it on controlled recipients, and keep a non-SMS recovery path available.

## Compare the operating models before procurement

The useful comparison axis is operational ownership. Score every candidate against the same route matrix and failure exercise; don't let one demo use a friendly destination while another is tested against the hard one.

| Operating model | What the team controls | Main trade-off | Suitable when |
| --- | --- | --- | --- |
| One transport behind a small adapter | Authentication policy and one provider contract | A route or contract change creates a concentrated migration | The destination set is narrow and the approved route is stable |
| Configurable adapter with approved routes | Per-destination identity choice and transport switching | The team owns normalization, configuration review, and a larger test matrix | Several markets have materially different operating requirements |
| General messaging abstraction | One application-facing contract across channels or transports | Normalization can conceal identity and delivery details needed during an incident | The team can inspect underlying route evidence and retain channel-specific controls |
| Regional implementations | Local operating knowledge and escalation | More credentials, contracts, adapters, and on-call relationships | Traffic is concentrated enough for local ownership to justify the overhead |

Do not score “number of countries” as a proxy for readiness. Score whether the exact sender and route can be registered, tested, observed, and supported. Do not score “has callbacks” without inspecting identifiers, authentication, duplicate behavior, ordering expectations, retention, and the process for an unknown event. Cost belongs in the evaluation as total messages per completed verification, including controlled retries and recovery; price cannot repair weak sender ownership or missing evidence.

Run a production-shaped pilot with opted-in, controlled recipients on the destinations available to the team. Exercise a first send, a permitted resend, a duplicate application request, an expired challenge, an old-generation code, duplicate status events, and recovery. Capture correlation IDs and timestamps at each boundary. Your mileage may vary by destination and user population, which is exactly why a small test on one convenient network can't stand in for the route matrix.

The catch is staffing. A configurable multi-route design is not suitable when nobody owns policy updates, registration renewals, callback mapping, and periodic delivery checks. In that case, stick with the smallest approved integration the team can operate and document the migration boundary. Regional implementations are also a bad default for a small footprint; they become reasonable when local registration and escalation work dominates the central team's capacity.

## Put the critical path behind a Python boundary

The following sketch keeps the decisive rule in the application: reserve a generation once, then dispatch through an interface. `reserve_dispatch` must be an atomic database operation in a real implementation. A losing worker returns the existing dispatch state instead of sending again.

```python
from dataclasses import dataclass
from enum import Enum
from typing import Protocol


class ReserveResult(Enum):
    CREATED = "created"
    EXISTS = "exists"


@dataclass(frozen=True)
class DispatchCommand:
    attempt_id: str
    generation: int
    destination: str
    sender_profile: str
    body: str

    @property
    def idempotency_key(self) -> str:
        return f"{self.attempt_id}:{self.generation}"


class DispatchStore(Protocol):
    def reserve_dispatch(self, key: str) -> ReserveResult:
        """Atomically reserve one dispatch for this code generation."""

    def attach_message_id(self, key: str, message_id: str) -> None:
        """Persist the transport identifier without exposing the OTP."""


class SmsTransport(Protocol):
    def send(self, command: DispatchCommand) -> str:
        """Return the transport's stable message identifier."""


def dispatch_once(
    command: DispatchCommand,
    store: DispatchStore,
    transport: SmsTransport,
) -> bool:
    result = store.reserve_dispatch(command.idempotency_key)
    if result is ReserveResult.EXISTS:
        return False

    message_id = transport.send(command)
    store.attach_message_id(command.idempotency_key, message_id)
    return True
```

This boundary is intentionally boring. Country policy is resolved before the command reaches it, so `sender_profile` names an approved configuration rather than accepting an arbitrary label from the client. Code generation and verification aren't shown because mixing them into a transport sample would blur ownership. Production code also needs transaction recovery, rate limiting, credential management, redaction, and authenticated event ingestion, all tested against the same state machine.

For observability, count attempts, authorized dispatches, transport-state transitions, successful verifications, expirations, and recovery starts as separate events. Alert on ratios that express user harm, such as a change in completed verification relative to authorized dispatch, then segment by destination and approved sender profile. Raw volume by itself can't tell an abuse spike from a product launch.

Email recovery remains separate. Public sender guidance from Yahoo documents operational requirements and best practices for email senders, while the linked email API documentation illustrates a distinct integration surface. That is enough to reject the tempting assumption that “we already send SMS” makes email fallback free of sender and deliverability work. Review, test, and observe it on its own terms.

## The rejected default still has a valid use case

The rejected option is putting a provider SDK directly in the login handler and letting its response determine challenge state. It couples authentication rules to transport behavior, makes duplicate browser requests harder to contain, and leaves sender selection scattered through application code. Swapping a route then becomes a login change — precisely the boundary this decision is meant to avoid.

Direct integration is still valid for a tightly scoped prototype with no real users, no production credentials, and no claim of launch readiness. It can answer whether the team understands the basic request and response shape. Before production, move it behind the adapter, make dispatch reservation durable, complete destination-level review, prove sender registration, authenticate status events, and test recovery. Easy setup is a useful experiment criterion. It is not the production selection criterion.

The final decision should be recorded per destination, not once for an entire continent. Approve the route whose evidence survives the registration, state-machine, security, deliverability, and support checks your team can actually operate. If none does, delay that SMS route and ship an already approved authentication method instead.

## References

- https://resend.com/docs/introduction
- https://senders.yahooinc.com/best-practices/
