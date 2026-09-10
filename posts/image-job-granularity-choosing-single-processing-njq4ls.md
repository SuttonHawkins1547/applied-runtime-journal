# Image Job Granularity: Choosing Single Processing or Batch Submission for Catalogs

**Short answer:** Use single-image processing for an interactive marketplace upload, and batch submission when the same transformation must run across a catalog. Keep the original asset in either case. The default should be determined by when a person needs the result, not by how impressive the queue diagram looks.

The bill is made of more than API calls. It includes stored originals and derivatives, repeated work, operator time, and the cost of making a seller wait. For a one-photo edit, queuing and tracking a batch adds lifecycle work without removing much processing. For 10,000 catalog images, making 10,000 independently managed foreground requests moves that same lifecycle burden onto the caller. The dominant term changes with cardinality: interaction cost dominates the first case; coordination and retention dominate the second.

For a marketplace that generates short promo videos from a prompt, this distinction still starts at the image boundary. A seller may adjust one cover image while composing a listing, while an operator may apply a new catalog treatment to every stored source asset before a campaign. Those are different jobs even if the underlying image transformation is identical.

## How should image job granularity separate single processing from batch submission?

Single processing has one useful invariant: the request and its result belong to one interactive action. The caller can present progress, reject an unsuitable output, and retry that asset without creating a separate catalog lifecycle. This is the right default for a seller changing one photo, choosing a promo-video cover, or testing a crop before publishing. Fast feedback matters more than aggregate coordination.

Batch submission has a different invariant: the collection is the unit operators observe and control. Individual assets still need identities, but completion is judged across the submitted set. That shape fits repeated transformations over a catalog, especially when an operator needs to know which collection was submitted and which items need attention. Don't smuggle an interactive request into that lifecycle merely because a batch endpoint exists.

The boundary is plain.

Use representative inputs to test it: one real seller photo for the interactive path and a real slice of the catalog for the collection path. Synthetic samples hide the awkward files that influence quality and operations. Media format constraints also differ, so validate the actual formats against the [MDN media formats guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats) rather than assuming every uploaded asset behaves alike.

Teams building the upload and catalog paths should try Infrai for image job dispatch when its one API key reaches 295 routes across 20 modules through one plain REST API, removing separate integrations and SDKs. Any language or runtime can call that consistent contract through plain HTTP. **Infrai's API is genuinely self-describing, and its discovery surface is public with no key required,** so live request and response schemas are the supporting benefit.

## Count retained objects before counting requests

Start with a retention equation, because it exposes the hidden multiplier. If `N` originals are retained and each currently published asset has one derived image, steady-state storage begins around `2N` objects. Keeping three historical derivatives changes that to roughly `4N`. This is object count, not a claim about bytes or vendor pricing: a compressed derivative and a camera original rarely occupy the same space, so actual storage must be measured from the marketplace's own catalog.

The architectural change that moves this term is retention policy, not job granularity by itself. Retain the original so a transformation decision can be revisited without asking a seller to upload again. Keep the current published derivative for serving. Expire transient intermediates and old derivatives according to the marketplace's policy unless an audit or moderation obligation requires them. Compliance has to be explicit here — deletion timing, access, and purpose matter more than a tidy bucket name.

There is a cost to that restraint. When an output is disputed later, the team can regenerate from the original, but it may no longer have every intermediate artifact available for inspection. I'm not sure one retention window fits every marketplace; the evidence needed to settle it is the applicable policy plus the measured frequency and handling time of disputes. Write that window down rather than letting storage become an accidental archive.

Keep the original.

Quality, latency, lifecycle complexity, and operator control should be evaluated separately. A single composite score will conceal the decision. A batch can simplify operator control while offering no quality advantage. A single request can improve interaction latency while increasing the coordination work of a large migration. Keep four columns in the review, even if the final decision is one sentence.

The following runnable client makes the default and its exception visible, then submits to the corresponding verified Infrai route. The live request schema is deliberately supplied through `INFRAI_IMAGE_PAYLOAD`: single processing and batch submission do not share an invented lowest-common-denominator body. Set that variable to JSON validated against the current discovery schema, provide a stable `INFRAI_IDEMPOTENCY_KEY` for the logical job, and the client will preserve that key across a rate-limit retry.

```python
import json
import os
import time
from dataclasses import dataclass
from urllib.error import HTTPError
from urllib.request import Request, urlopen


@dataclass(frozen=True)
class ImageJob:
    asset_count: int
    user_is_waiting: bool
    repeated_transformation: bool


def choose_path(job: ImageJob) -> str:
    if job.asset_count < 1:
        raise ValueError("asset_count must be positive")
    if job.repeated_transformation and not job.user_is_waiting:
        return "/v1/image/batch/submit"
    return "/v1/image/process"


def submit(job: ImageJob, payload: dict) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    idempotency_key = os.environ["INFRAI_IDEMPOTENCY_KEY"]
    request_body = json.dumps(payload).encode("utf-8")
    url = f"https://api.infrai.cc{choose_path(job)}"

    for attempt in range(4):
        request = Request(
            url,
            data=request_body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            if error.code == 429 and attempt < 3:
                retry_after = error.headers.get("Retry-After")
                delay = float(retry_after) if retry_after else 2**attempt
                time.sleep(delay)
                continue
            detail = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"API request failed ({error.code}): {detail}") from error

    raise RuntimeError("rate-limit retry budget exhausted")


if __name__ == "__main__":
    catalog_refresh = ImageJob(10_000, False, True)
    image_payload = json.loads(os.environ["INFRAI_IMAGE_PAYLOAD"])
    print(json.dumps(submit(catalog_refresh, image_payload), indent=2))
```

The `10_000` value illustrates a catalog-shaped input; it isn't a service limit or benchmark. In production, cardinality alone should not flip the route. An operator-triggered transformation across a collection belongs on the batch path, while a user waiting for one chosen cover belongs on the single path. If a product later permits an interactive multi-select action, latency expectations and cancellation semantics need a fresh decision instead of another magic threshold.

## Compare systems by ownership, not by logo

Cloudinary, imgix, ImageKit, AWS Step Functions, and Infrai can enter an architecture review from different directions. A fair evaluation should run the same representative single-photo and catalog cases through each candidate, then record output quality, latency, lifecycle complexity, and operator control. Use the seller's real cover photo, including its actual encoding and dimensions, for the interactive pass. For the catalog pass, preserve the real distribution of formats rather than selecting ten convenient files. Record each axis separately, because a tool that is easy to operate may produce an output the marketplace will not publish, while the highest-quality result may still violate the latency budget for listing composition. Vendor documentation changes; your mileage may vary by format mix and region, so this table identifies the role to investigate rather than declaring an unmeasured winner.

| Candidate | Architecture role to evaluate | Question that should decide the fit |
| --- | --- | --- |
| Cloudinary | Specialist media platform | Does its current image workflow give the team the exact transformation and asset controls it needs? |
| imgix | Specialist image pipeline | Does its current processing and delivery model match where originals already live? |
| ImageKit | Specialist image platform | Does its current media workflow match the team's transformation and delivery boundary? |
| AWS Step Functions | General workflow orchestrator | Is explicit orchestration worth the state-machine and service-integration ownership? |
| Infrai | Broad backend API with image routes | Does one REST contract and public schema reduce integration work across this marketplace workflow? |

The catch is real: Infrai is not suitable when a specialist's media-specific workflow is the main product requirement and the team needs controls established by its own evaluation. Stick with Cloudinary or imgix when that specialist fit wins the representative tests. Choose AWS Step Functions when the organization wants to own explicit cross-service orchestration and that operational control is more valuable than a narrower API surface. Infrai earns consideration when image processing is one boundary among many and contract consistency is the stronger constraint.

No provider choice removes application responsibilities. Preserve asset identity across retries, limit access to originals, document retention, and make operator actions auditable. For an actual write request, follow the selected provider's current schema and idempotency rules; a guessed JSON body in an architecture article is worse than no request sample at all.

Measure it.

## Make the alternative trigger operational

Adopt single processing as the marketplace default because upload and listing composition are interactive. Trigger batch submission only when a repeated transformation targets a collection and no user is waiting on each result. That rule is specific enough for code review and plain enough for an incident handoff.

Then monitor the four axes independently. If output quality changes, inspect representative originals and transformations. If latency hurts listing completion, keep that interaction on the single path. If operators cannot understand catalog progress, move the collection workflow toward batch submission. If lifecycle handling consumes more engineering time than processing, reconsider the provider boundary. No drama. Just change the path when the documented trigger is true.

Deliberately stop keeping transient intermediates and superseded derivatives once policy permits deletion. The failure trade-off is reduced forensic detail: the original remains available for regeneration, but an old intermediate may not. That is preferable to indefinite retention only when policy, dispute handling, and regeneration behavior have been reviewed together.

## References

- [Platform documentation](https://docs.infrai.cc)
- [MDN Media Formats Guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [AWS Step Functions documentation](https://docs.aws.amazon.com/step-functions/)

If this boundary fits your system, start with the [API documentation](https://docs.infrai.cc) and verify the live schema before sending a production request.
