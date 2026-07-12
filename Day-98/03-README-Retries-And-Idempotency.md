# Day 98 — Toil Elimination & Reliability Patterns: Retry Strategies & Idempotency

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** SRE

## Brief

Retrying a failed request sounds like an obviously good idea — and done wrong, it's one of the most common ways engineers accidentally *cause* an outage rather than survive one. Naive retries can amplify load on an already-struggling dependency (a "retry storm") and, if the underlying operation wasn't safe to repeat, can silently duplicate side effects (double-charging a customer, sending a duplicate email, creating two orders from one checkout click). This file covers how to retry safely — exponential backoff with jitter — and the property that makes retries (and circuit-breaker fallbacks, and at-least-once message delivery) safe in the first place: **idempotency**.

## Why naive retries make things worse: the retry storm

If every client, on failure, immediately retries with the same fixed delay (or no delay), and many clients fail at roughly the same time (because the shared dependency they're all calling just became slow/unhealthy), all those retries arrive **simultaneously and repeatedly**, adding *more* load onto a dependency that's already struggling — often enough additional load to prevent it from ever recovering. This is called a **retry storm**, and it's a well-documented real-world cause of outages lasting far longer than the original triggering event, because the retries themselves become the sustaining cause of the outage.

## Exponential backoff with jitter

**Exponential backoff**: instead of retrying immediately, wait progressively longer between attempts — `base_delay * 2^attempt` (e.g., 1s, 2s, 4s, 8s, 16s...), capped at some maximum delay. This gives a struggling dependency progressively more breathing room to recover instead of being retried at a constant, possibly overwhelming, rate.

**Jitter**: add randomness to the delay so that many clients who failed at the same moment don't all retry at exactly the same computed delay (which would just recreate a synchronized retry storm, only spaced out in discrete waves instead of continuously). Without jitter, exponential backoff alone still produces thundering-herd retry spikes at each backoff interval — jitter smears them out over time.

```python
import random
import time

def call_with_retry(fn, max_attempts=5, base_delay=1, max_delay=30):
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError:
            if attempt == max_attempts - 1:
                raise
            # "Full jitter" (AWS's recommended formula): random between 0 and the
            # exponential cap, not just +/- a small percentage around it.
            delay = random.uniform(0, min(max_delay, base_delay * (2 ** attempt)))
            time.sleep(delay)
```

Using **Tenacity** (this day's assigned Python retry library), the same logic is declarative:

```python
from tenacity import retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception_type

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=1, max=30),
    retry=retry_if_exception_type(TransientError),
)
def call_payment_gateway(order):
    return requests.post("https://payment-gateway/charge", json=order, timeout=2)
```

Key design decisions that matter, beyond just "add backoff and jitter":

- **Only retry transient/retriable failures** — a network timeout or a 503/429 response is worth retrying; a 400 (bad request) or 401 (unauthorized) will fail identically every time and retrying it just wastes time and load for no benefit. `retry_if_exception_type` (or checking the response status code) should be scoped deliberately, not "retry on any exception."
- **Cap the total number of attempts and the total elapsed time**, not just the per-attempt delay — an unbounded retry loop that never gives up is itself a resource leak and can keep a request "in flight" from the user's perspective indefinitely.
- **Respect a `Retry-After` header if the dependency provides one** (common on 429 rate-limit responses) — the dependency is telling you exactly how long to wait; ignoring it in favor of your own backoff schedule can still contribute to overload.
- **Combine with a circuit breaker (file 2)** — once a breaker has tripped open, stop retrying against that dependency entirely rather than continuing to retry-with-backoff against a call that will fail-fast immediately anyway; retries and circuit breakers should compose, with the breaker taking precedence once open.

## Idempotency: what makes retries (and at-least-once delivery) safe

**Idempotency** means: performing the same operation multiple times has the **same effect as performing it once**. This property is what makes retries safe in the first place — if `charge_customer(order_id)` is idempotent, retrying it after a timeout (where you genuinely don't know if the first attempt succeeded or failed before you got the response) is safe, because even if the first attempt *did* actually succeed server-side and you just didn't get the response, the retry won't double-charge the customer.

**Why this matters more than it seems:** a request can fail from the *caller's* perspective (timeout, connection reset) while still having **succeeded** on the *server's* side — the server processed it and the response was lost in transit, or the server processed it and then crashed before sending the response. The caller has no way to distinguish "it never happened" from "it happened, I just didn't hear back" — which is exactly why blind retries without idempotency are dangerous: you cannot tell whether a retry is completing an unfinished operation or duplicating a completed one.

### The standard mechanism: idempotency keys

The caller generates a unique key (usually a UUID) **once**, before the first attempt, and sends it with every retry of the *same logical operation*:

```python
import uuid
import requests

idempotency_key = str(uuid.uuid4())  # generated ONCE, reused across retries

def charge(order):
    return requests.post(
        "https://payment-gateway/charge",
        json=order,
        headers={"Idempotency-Key": idempotency_key},
        timeout=2,
    )
```

On the server side, before processing a request, it checks: "have I already seen this idempotency key?" If yes, it returns the **stored result of the original processing** (without re-executing the side effect — no second charge, no second order created); if no, it processes the request normally and stores the key + result for some retention window (long enough to cover realistic retry/timeout windows, e.g., 24 hours). This is exactly how Stripe's and most major payment APIs' idempotency-key mechanisms work, and it's the pattern to reach for any time you're designing an API that might be retried (which, in a distributed system, is effectively always).

### Idempotency without an explicit key: natural idempotency

Some operations are idempotent by their *nature*, with no extra key needed:
- `SET x = 5` is idempotent — running it twice leaves `x` at 5 either way.
- `INCREMENT x BY 1` is **not** idempotent — running it twice changes the result.
- `DELETE resource_id` is typically idempotent — deleting an already-deleted resource is a no-op (as long as the API returns success/404-treated-as-success rather than erroring on "not found").
- `INSERT` without a uniqueness constraint is **not** idempotent (creates duplicates); `INSERT ... ON CONFLICT DO NOTHING` (or an upsert keyed on a natural unique identifier, like `order_id`) makes it idempotent.

**Designing for idempotency from the start** (using natural keys, upserts, or explicit idempotency-key mechanisms) is far cheaper than retrofitting it after a production double-charge incident — this is precisely the kind of design decision that determines whether "just add retries" is safe or dangerous for a given operation.

## Points to Remember

- Naive retries (no backoff, no jitter, synchronized across many clients) can create a retry storm that sustains or worsens an outage rather than helping recover from it.
- Exponential backoff spaces out retries progressively; jitter (randomizing the delay) prevents synchronized retry waves even with backoff in place — use both together (Tenacity's `wait_exponential_jitter` does this out of the box).
- Only retry transient/retriable failures (timeouts, 503/429), never non-retriable ones (400, 401/403) — and always cap total attempts/elapsed time.
- Idempotency = repeating an operation has the same effect as doing it once — this is the property that makes retries (and at-least-once message delivery generally) safe rather than dangerous.
- The standard mechanism is a client-generated idempotency key, sent unchanged across retries of the same logical operation, which the server uses to detect and short-circuit duplicate processing while still returning the original result.

## Common Mistakes

- Retrying immediately with no backoff (or fixed, non-jittered backoff) against a struggling dependency — turning a brief blip into a sustained retry storm that prevents the dependency from ever recovering.
- Retrying non-idempotent operations (e.g., "create order," "charge card") without an idempotency-key mechanism, causing duplicate side effects (double charges, duplicate orders/emails) whenever a request times out after actually succeeding server-side.
- Retrying on every exception type indiscriminately, including permanent failures (bad request, auth failure) that will never succeed no matter how many times you retry — wasting time/load for zero benefit and delaying the eventual (inevitable) failure response to the user.
- Not capping total retry attempts or total elapsed retry time, leaving requests "hanging" from the user's perspective far longer than a clean, fast failure would have.
- Assuming a circuit breaker alone prevents retry storms — a retry loop layered *inside* each individual call, with the breaker only wrapping the outermost call, can still hammer a dependency with retries per-call before the breaker ever sees enough failures to trip; retries and breakers need to be composed deliberately, with clear precedence (stop retrying once the breaker is open).
