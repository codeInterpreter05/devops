# Day 98 — Resources: Toil Elimination & Reliability Patterns

## Primary (assigned)

- **Google SRE Book — Chapter 5: Eliminating Toil** (sre.google/sre-book/eliminating-toil, free online) — the assigned starting point for this day, and the original source for the precise, six-property definition of toil used throughout this day's notes.

## Deepen your understanding

- **Google SRE Book — Chapter 6: Monitoring Distributed Systems** and **Chapter 21: Managing Critical State** (sre.google/sre-book) — useful adjacent chapters connecting toil reduction to broader reliability engineering practice.
- **Netflix Technology Blog — "Fault Tolerance in a High Volume, Distributed System"** — the original Hystrix-era writeup explaining circuit breakers, bulkheads, and the cascading-failure problem from the team that popularized these patterns at scale.
- **Resilience4j documentation** (resilience4j.readme.io) — the modern, actively-maintained Java resilience library (Hystrix's spiritual successor); its docs are an excellent, concrete reference for circuit breaker, bulkhead, retry, and rate-limiter configuration even if you're implementing equivalents in Python.
- **Tenacity documentation** (tenacity.readthedocs.io) — this day's assigned Python retry library; covers `wait_exponential_jitter`, stop conditions, and retry-condition predicates used in today's lab.
- **AWS Architecture Blog — "Exponential Backoff And Jitter"** — the widely-cited post that introduced "full jitter" as the recommended backoff randomization strategy, with benchmark data showing why plain exponential backoff alone isn't sufficient.
- **Stripe API documentation — Idempotent Requests** (stripe.com/docs/api/idempotent_requests) — the industry-reference implementation of idempotency keys in a real, widely-used payment API; worth reading to see the pattern used in file 3 applied at production scale.

## Reference

- **Martin Fowler — "CircuitBreaker"** (martinfowler.com/bliki/CircuitBreaker.html) — a concise, widely-cited explanation of the pattern and its motivating cascading-failure scenario.
- **Resilience4j GitHub repo README** (github.com/resilience4j/resilience4j) — quick side-by-side comparison of all the resilience patterns (circuit breaker, bulkhead, retry, rate limiter, time limiter) it implements, useful as a checklist of what a mature resilience layer covers.

## Practice

- **Today's lab** (identify/automate real toil, then build the flaky-service + Tenacity retry-storm demonstration) — the most direct way to internalize both halves of this day's material with your own hands and your own numbers.
- **Chaos-engineering-style local experiments** — once comfortable, try intentionally injecting failures (kill a dependency mid-request, add artificial latency) into a personal project and observing whether your retry/circuit-breaker/bulkhead setup behaves as designed — the best way to build real confidence in these patterns before relying on them in production.
