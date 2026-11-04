# Logistics Tenant Subdomain Cutovers: 2 CNAME Ownership Boundaries and the Apex Limitation

Keep the logistics tracking hostname on a customer-controlled subdomain, point it at a platform target with a CNAME, and retain the previous destination through an explicitly bounded rollback window. That is the least complex starting point when customers need to keep their own names. The bill includes overlap between old and new serving capacity, DNS administration, validation work, and retained operational evidence. Before comparing record types, quantify the overlap: retaining one old destination alongside one new destination means two serving destinations for the transition. This is an illustrative capacity count, not a measured cost estimate; their actual charges depend on capacity and duration.

TL;DR: Use CNAME for a tenant subdomain when the platform should control changes behind a stable target name. Use A records when the name must point directly to IPv4 addresses, including a conventional zone apex where CNAME conflicts with required zone data. Customer-owned versus platform-owned zones determine who can execute rollback. Record type alone cannot grant that authority.

The change that reduces overlap is retiring the old destination after the rollback window closes. Lowering a DNS TTL does not retire it. This note treats the cutover as an evidence-retention decision: what must remain available to justify that retirement, and what evidence can be discarded without collecting parcel recipients' delivery details?

## Who can reverse the tracking hostname change?

Consider an illustrative parcel-tracking service with a customer-owned name, track.shipper.example, and a platform-owned destination, tenant-42.edge.example. These are example names, not live endpoints. With a CNAME at the customer name, the customer controls the alias while the platform controls resolution behind the target. If the platform can restore the old service through that same target, it can reverse a deployment without asking the customer to edit DNS. If reversal requires replacing the customer's alias, the customer must participate. Write that distinction into the release plan before scheduling the cutover.

Authority comes first.

The trade-off is operational control: customer-owned DNS is not suitable for a release plan that requires the platform to edit customer records without authorization. Use a platform-owned hostname if that control is a requirement and the customer accepts the naming boundary.

For a platform-owned zone, the release team can operate tenant names directly under its own namespace. That reduces coordination for record edits, but the customer no longer controls that hostname's DNS lifecycle. For a customer-owned zone, require an authorized operator and record the approved scope of the change. Control of an application tenant is not, by itself, permission to change a customer's zone. Where authority is delegated, document the actual zone boundary; a subdomain label does not automatically create a separately delegated zone. These distinctions follow the zone and delegation model in RFC 1034.

A logistics rollback also needs an application boundary. Restoring name resolution does not reverse shipment updates already accepted by the new service. Define success as preserving tracking access and the correct tenant association, then handle any application-state reconciliation separately. This is a proposed operating rule, not a property supplied by DNS. The old destination must still have the required application configuration if it is meant to receive traffic again.

## CNAME versus A records, including the apex constraint

RFC 1035 defines an A record as an IPv4 address and a CNAME as a canonical name. For the example tracking subdomain, a CNAME lets the customer publish the platform's target name rather than maintain the target's address set. The platform can then manage that destination's records. This reduces customer coordination when addresses change, at the cost of depending on the target's continued resolution and on a clear lifecycle for that target. An A record exposes an address directly; changing the serving address then requires updating that record wherever it is controlled.

An A record isn't a complete dual-stack configuration: IPv6 addresses use AAAA records, defined in RFC 3596. If both families are published, include both in the cutover evidence. Otherwise, a successful IPv4 check can conceal a different IPv6 destination.

Check both families.

The apex restriction concerns the apex of a DNS zone, not merely a short-looking hostname. A zone apex has SOA and NS data. RFC 2181's CNAME rules prohibit ordinary data from coexisting with a CNAME at the same name, so a conventional apex cannot be a CNAME. A delegated subdomain has its own apex and encounters the same constraint. Use suitable address records there, or assess an authoritative service's explicitly documented alias-synthesis behavior as a separate capability. Don't assume that capability is a portable DNS record type.

There is a second, smaller trap: putting a CNAME at a name that already needs ordinary TXT or MX data. Check the entire owner name before proposing the alias. Moving a web tracking name should not quietly rewrite a customer's mail domain. DMARC, specified in RFC 7489, evaluates email authentication and alignment; making the tracking URL resolve proves nothing about OTP email authentication or inbox placement. Keep the acceptance checks separate.

## Keep a rollback manifest, then test the two destinations

Use a small manifest to preserve the control boundary and the previous destination. The following Python example generates a reviewable plan for the illustrative tracking name. It deliberately returns work for an authorized operator; it does not perform DNS edits, discover ownership, or prove that a destination is serving the right tenant. An explicit authorization check prevents a plan from presenting an unapproved customer-zone change as ready to execute.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Cutover:
    hostname: str
    zone_owner: str
    record_type: str
    previous_values: tuple[str, ...]
    proposed_values: tuple[str, ...]
    authorized: bool


def rollback_plan(change: Cutover) -> dict[str, object]:
    if change.zone_owner not in {"customer", "platform"}:
        raise ValueError("Unknown zone owner")
    if not change.authorized:
        raise PermissionError("Zone change authorization is required")
    if change.record_type not in {"A", "AAAA", "CNAME"}:
        raise ValueError("Unsupported record type for this manifest")
    if not change.previous_values or not change.proposed_values:
        raise ValueError("Both record sets are required")
    if change.record_type == "CNAME" and (
        len(change.previous_values) != 1
        or len(change.proposed_values) != 1
    ):
        raise ValueError("A CNAME must have one target")
    return {
        "hostname": change.hostname,
        "executor": change.zone_owner,
        "record_type": change.record_type,
        "restore": change.previous_values,
        "replace": change.proposed_values,
    }


change = Cutover(
    hostname="track.shipper.example",
    zone_owner="customer",
    record_type="CNAME",
    previous_values=("tenant-42.old.example",),
    proposed_values=("tenant-42.edge.example",),
    authorized=True,
)
plan = rollback_plan(change)
```

This manifest has a narrow purpose. A production change record should also preserve the observed TTL, approval reference, change time, and validation outcome under the team's access and retention policy. Record the previous state before applying the new one. If the record changes again between review and execution, stop and reconcile the discrepancy instead of overwriting someone else's update. The code does not implement that concurrency check.

Test both destinations with the intended hostname and verify certificate validity, tenant routing, and a synthetic tracking response before switching. Keep actual parcel identifiers and OTP values out of those synthetic checks. During the overlap, observe resolution and application outcomes separately: reaching an address is weaker evidence than receiving the intended tenant's response. If the application redirects, inspect the destination too. A correct initial hostname can still send the recipient elsewhere.

DNS caching makes rollback a transition rather than a single instant. RFC 1034 describes TTL-based caching; changing the authoritative record does not erase already cached answers. A lower TTL published immediately before the cutover does not shorten the lifetime of an answer cached earlier under a higher TTL. Negative answers also have caching rules, described in RFC 2308, so test provisioning before customers distribute a new tracking link. A single resolver's answer is insufficient evidence that all relevant clients have moved.

## Close the window by deleting evidence deliberately

Set the rollback window from the actual cache behavior and application acceptance criteria your team can verify. There is no universal duration established by these DNS standards. During that window, retain the previous record set, an authorized rollback executor, a usable old destination, and compact validation evidence. Continuing to run both destinations indefinitely preserves an option, but also extends capacity overlap and the responsibility to maintain the old tenant configuration.

Retirement is a decision.

Keep its approval.

At closure, remove the old serving configuration through the normal deprovisioning process and retain the minimal change record required by the applicable policy. For this workflow, deliberately stop retaining raw tracking query strings, parcel payloads, recipient contact details, and OTP contents as cutover evidence; synthetic outcomes and restricted change metadata are the proposed substitute. Choose retention periods with the relevant operational and compliance owners rather than inventing a universal legal deadline.

The cost of that choice is reduced forensic detail. If a recipient later reports that one tracking request reached the wrong tenant during the transition, a synthetic success record cannot reconstruct that exact response. Accept that limit explicitly, or approve narrowly scoped evidence collection before the change. The useful endpoint is a tracking hostname with an identified DNS owner, a tested reversal path while the window is open, and a documented reason for retiring the old destination when it closes.

## References

The DNS record definitions, zone rules, and caching behavior above are grounded in the standards listed below. The logistics names, capacity count, manifest, and retention policy are illustrative engineering choices rather than measured deployments. RFC 7489 supports the separate email-authentication boundary; it does not promise OTP delivery.

## Further reading

- RFC 1034 — Domain Names: Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035 — Domain Names: Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 2181 — Clarifications to the DNS Specification: https://datatracker.ietf.org/doc/html/rfc2181
- RFC 2308 — Negative Caching of DNS Queries: https://datatracker.ietf.org/doc/html/rfc2308
- RFC 3596 — DNS Extensions to Support IP Version 6: https://datatracker.ietf.org/doc/html/rfc3596
- RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
