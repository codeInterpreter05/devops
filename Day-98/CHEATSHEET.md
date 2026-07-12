# Day 98 — Cheatsheet: Toil Elimination & Reliability Patterns

## Toil — the six-property test

```
Manual            - a human has to do it
Repetitive        - happens over and over
Automatable       - a machine could do it instead
Tactical          - reactive/interrupt-driven, not strategic
No enduring value - system is same after as before
Scales w/ growth  - grows with traffic/service count, not flat

Sharpest test: does this leave the system durably better, or recur unchanged next week?
SRE classic cap: toil < 50% of an SRE's time.
```

## Circuit breaker states

```
CLOSED    -> normal traffic flows, breaker watches failure rate
   |  failure threshold exceeded (e.g. >50% of last 20 calls failed)
   v
OPEN      -> fail FAST, no calls to dependency, for cooldown period
   |  cooldown expires
   v
HALF-OPEN -> allow a few trial requests
   |-- success --> CLOSED
   |-- failure --> OPEN (wait another cooldown)
```

```python
import pybreaker
breaker = pybreaker.CircuitBreaker(fail_max=5, reset_timeout=30)

@breaker
def call_dependency():
    return requests.post(url, timeout=2)
```

## Bulkhead pattern

```
BAD:  one shared thread pool for ALL outbound calls
      -> one slow dependency exhausts the pool -> everything else starves too

GOOD: dedicated pool PER dependency (or per criticality tier)
      -> one slow dependency only exhausts ITS OWN pool
      -> everything else keeps working
```

## Retry with exponential backoff + jitter (Tenacity)

```python
from tenacity import retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception_type

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=1, max=30),
    retry=retry_if_exception_type(TransientError),
)
def call():
    ...
```

```
Backoff:  delay = base * 2^attempt   (1s, 2s, 4s, 8s, 16s... capped)
Jitter:   delay = random(0, backoff_cap)   ("full jitter" - AWS recommended)
Only retry: timeouts, 503, 429 (respect Retry-After header)
Never retry: 400, 401, 403 (will fail identically every time)
Always cap: max attempts AND max total elapsed time
```

## Idempotency

```
Idempotent:      repeating it = same effect as doing it once
SET x = 5        -> idempotent
INCREMENT x +=1  -> NOT idempotent
DELETE id        -> idempotent (no-op if already deleted)
INSERT (no key)  -> NOT idempotent (creates dupes)
UPSERT on natural key / ON CONFLICT DO NOTHING -> idempotent
```

```python
import uuid
idempotency_key = str(uuid.uuid4())   # generate ONCE, reuse across every retry

requests.post(url, json=payload, headers={"Idempotency-Key": idempotency_key})
```

Server side: check key seen before -> return STORED original result, don't reprocess.

## Composing the patterns (typical stack, outer to inner)

```
Circuit Breaker (stop calling a known-dead dependency)
  -> Retry w/ backoff+jitter (handle transient blips)
    -> Bulkhead (isolated resource pool per dependency)
      -> Timeout (bound how long any single attempt can take)
        -> Idempotency key (make retries/duplicates safe)
```

## Avoiding retry storms

```
Naive retry (no backoff, synchronized) -> load spike concentrated -> sustains outage
Backoff + jitter -> retries spread out over time -> dependency gets room to recover
```
