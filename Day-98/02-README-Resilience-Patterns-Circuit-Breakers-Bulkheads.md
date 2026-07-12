# Day 98 — Toil Elimination & Reliability Patterns: Circuit Breakers & the Bulkhead Pattern

**Phase:** 3 – Observability | **Week:** W16 | **Domain:** SRE

## Brief

In a distributed system, the most common cause of a *small* problem becoming a *large* outage isn't the original failure itself — it's the failure **cascading** through every service that depends on the failing one, each piling on retries and holding resources open while waiting for a response that isn't coming. Circuit breakers and bulkheads are the two classic patterns (popularized by Netflix's Hystrix library, and now embodied in libraries like Resilience4j) for containing that cascade — stopping one unhealthy dependency from taking down everything that talks to it. Understanding both, precisely, is what separates "we added retries" (often makes cascades *worse*, see file 3) from actually engineering for resilience.

## The cascading failure problem

Picture service A calling service B, which is slow or down. Without any protection:
1. Every request from A to B hangs waiting for a response (or a slow one), holding a thread/connection open the whole time.
2. A's thread pool / connection pool fills up with requests stuck waiting on B.
3. A now has no capacity left to serve *any* request — including ones that don't even touch B — because its resources are exhausted waiting on the unrelated slow dependency.
4. A becomes unresponsive to *its* callers, and the failure cascades one hop further upstream, repeating the same pattern.

This is exactly how a single database's slowness can, within minutes, take down an entire service mesh with no code bug anywhere except "nobody protected against a slow dependency." Circuit breakers and bulkheads exist specifically to stop step 2→3 from happening.

## Circuit breakers

A circuit breaker wraps a call to a dependency and tracks its recent success/failure rate. It has three states, modeled directly on an electrical circuit breaker:

- **Closed** (normal) — calls pass through to the dependency as usual; the breaker is watching the failure rate in the background.
- **Open** — once failures exceed a configured threshold (e.g., >50% of the last 20 calls failed, or 5 consecutive timeouts), the breaker "trips": it **stops calling the dependency entirely** for a configured cooldown period, immediately returning a fast failure (or a fallback) to the caller instead of attempting the call and waiting for it to time out.
- **Half-Open** — after the cooldown expires, the breaker allows a small number of trial requests through. If they succeed, it closes again (resumes normal traffic); if they still fail, it re-opens and waits another cooldown period.

```python
# Conceptual circuit breaker behavior (pybreaker-style)
import pybreaker

payment_breaker = pybreaker.CircuitBreaker(
    fail_max=5,          # trip after 5 consecutive failures
    reset_timeout=30,    # try again (half-open) after 30s
)

@payment_breaker
def call_payment_gateway(order):
    return requests.post("https://payment-gateway/charge", json=order, timeout=2)

try:
    call_payment_gateway(order)
except pybreaker.CircuitBreakerError:
    # Breaker is OPEN — fail fast instead of waiting on a timeout, use fallback
    queue_for_retry_later(order)
```

**Why "fail fast" is the whole point:** without a breaker, every caller still attempts the call and waits out the full timeout (say, 30 seconds) before giving up — multiplied across every concurrent request, this is exactly what exhausts A's thread pool in the cascade scenario above. With an open breaker, A knows *immediately* (no network round-trip, no waiting) that B is unhealthy, and can return a fallback or a fast error in milliseconds — preserving its own capacity to serve other requests, and giving the unhealthy dependency breathing room to recover instead of being hammered by continued traffic while already struggling.

**The half-open state matters as much as open/closed** — without it, you'd need either a fixed timer with no verification (risking resuming full traffic onto a still-broken dependency) or manual intervention to close the breaker again. Half-open lets the system self-heal: a small trickle of test traffic verifies recovery before fully reopening the floodgates.

## The bulkhead pattern

Named after ship design — a ship's hull is divided into watertight compartments (bulkheads) so that a hull breach in one compartment floods only that section, not the entire ship. Applied to software: **partition resources (thread pools, connection pools, or entire service instances) so that one dependency's failure can only exhaust the resources allocated to *it*, not the resources shared by everything else.**

Concretely:

```
Without bulkheads:
  Service A has ONE shared thread pool (size 100) for ALL outbound calls
  (payment-gateway, inventory-service, recommendation-service, ...)
  -> payment-gateway goes slow -> all 100 threads eventually stuck waiting on it
  -> inventory-service and recommendation-service calls can't get a thread either,
     even though THEY are perfectly healthy
  -> Service A is now down for everything, not just payment-related requests

With bulkheads:
  payment-gateway calls:        dedicated pool of 20 threads
  inventory-service calls:      dedicated pool of 30 threads
  recommendation-service calls: dedicated pool of 20 threads
  -> payment-gateway goes slow -> its 20 threads get stuck
  -> inventory/recommendation calls still have their own untouched pools
  -> Service A can still serve everything EXCEPT payment-related requests
```

Bulkheads are typically implemented via **per-dependency thread/connection pool isolation** (what Hystrix/Resilience4j provide directly), or at a coarser level via **separate service instances/deployments** per criticality tier (e.g., running the checkout path on separate infrastructure from an internal admin tool, so admin-tool load spikes can't degrade checkout). The core idea generalizes beyond thread pools: any shared, finite resource (connection pool, semaphore, even a Kubernetes node pool) can be bulkheaded by partitioning it per-dependency or per-criticality instead of pooling everything together.

**Circuit breakers and bulkheads are complementary, not substitutes:** a bulkhead limits the *blast radius* of a slow dependency (it can only exhaust its own partition); a circuit breaker limits the *duration/frequency* of wasted effort calling a known-unhealthy dependency. Production resilience libraries (Resilience4j, and historically Hystrix) implement both side by side, often composed with retry logic (file 3) and timeouts as a stacked set of protections around every outbound call.

## Points to Remember

- Cascading failure happens when a slow/failing dependency causes resource exhaustion (threads/connections stuck waiting) in the *calling* service, which then can't serve unrelated requests either — the failure spreads upstream one hop at a time.
- Circuit breaker states: Closed (normal, watching failure rate) → Open (fail fast, no calls to the dependency, for a cooldown) → Half-Open (trial requests to test recovery) → back to Closed or Open depending on result.
- The value of "open" is failing fast (milliseconds) instead of waiting out a full timeout on every call — this is what actually preserves the caller's own capacity during an incident.
- Bulkheads partition finite shared resources (thread pools, connection pools, even infrastructure) per-dependency or per-criticality, so one dependency's failure can only exhaust its own partition, not everything.
- Circuit breakers and bulkheads solve different, complementary problems (frequency/duration of wasted calls vs. blast radius of resource exhaustion) — production systems use both together, plus retries/timeouts (file 3).

## Common Mistakes

- Adding a circuit breaker but leaving a single shared thread pool for all outbound dependencies — the breaker stops *new* wasted calls once tripped, but doesn't prevent the initial resource exhaustion that got the pool stuck in the first place; bulkheads solve that separate problem.
- Setting circuit breaker thresholds so sensitive (e.g., trips after 1 failure) that transient, harmless blips trip the breaker constantly, causing the service to treat a healthy dependency as unhealthy far too often (a form of alert-fatigue equivalent, but for automated failover).
- Not implementing the half-open state (or its equivalent) at all — either the breaker never recovers automatically (needs manual reset, adding toil) or it resumes 100% of traffic the instant the cooldown timer expires without verifying recovery first, potentially re-triggering the same overload.
- Pooling all outbound dependency calls into one shared connection/thread pool "for simplicity," not realizing this creates a single point of cascading failure across otherwise-unrelated dependencies.
- Assuming a circuit breaker alone is sufficient resilience — without a sensible fallback behavior (return cached data, degrade gracefully, queue for later) for the open-circuit case, "fail fast" just means failing *immediately* instead of failing *slowly*, which is better but still a failure the user experiences if no fallback is implemented.
