# Day 98 — Lab: Toil Elimination & Reliability Patterns

**Goal:** Identify and quantify real toil from your own experience, automate one item and calculate time saved (the assigned hands-on activity), then build and break a small Python service demonstrating retries, backoff+jitter, and idempotency with Tenacity.

**Prerequisites:** Python 3.10+, `pip install tenacity flask requests`.

---

### Lab 1 — Identify and measure your top 3 toil items

1. From your internship/coursework/personal projects, list every recurring operational task you can recall from the last 1-3 months. For each, check it against all six toil properties (manual, repetitive, automatable, tactical, no enduring value, scales with growth) — cross out anything that fails even one property (e.g., requires real judgment, or was a one-time task).
2. For your top 3 remaining candidates, estimate: frequency (times per week/month) and time per occurrence (minutes). Multiply to get a rough monthly hours-spent figure for each.
3. Rank them by monthly hours. Write one sentence per item explaining *why* it's automatable (what would the automation need to check/do?).

**Success criteria:** A written, ranked list of 3 real toil items with a defensible hours/month estimate for each, and a one-line automation sketch for each — not vague ("stuff takes too long") but specific enough that someone else could start building the automation from your notes.

---

### Lab 2 — Automate one toil item and calculate time saved

1. Pick the highest-value (or most tractable, if time-boxed) item from Lab 1.
2. Build the smallest real automation that removes the manual step — a script, a scheduled job (cron/systemd timer), a Slack-bot command, a CI pipeline step, whatever fits. It doesn't need to be production-grade; it needs to actually replace the manual action for at least one real occurrence.
3. Run/trigger it for real at least once, confirming it does what the manual process did.
4. Calculate time saved: `(minutes per manual occurrence - minutes per automated occurrence) x occurrences per month`. Be honest about the time spent *maintaining* the automation itself — subtract a reasonable ongoing-maintenance estimate.

**Success criteria:** A working automation (however small) that you've run at least once, plus a specific "hours saved per month" number with your math shown, including the maintenance-cost subtraction.

---

### Lab 3 — Build a flaky dependency and observe retry storms

1. Create a minimal Flask "flaky service" that fails a configurable percentage of requests with a slow response:
   ```python
   # flaky_service.py
   from flask import Flask
   import time, random

   app = Flask(__name__)
   FAIL_RATE = 0.6

   @app.route("/charge", methods=["POST"])
   def charge():
       if random.random() < FAIL_RATE:
           time.sleep(2)
           return {"error": "timeout"}, 503
       return {"status": "charged"}, 200

   app.run(port=6000)
   ```
2. Write a naive client with NO backoff — immediate retry, fixed count:
   ```python
   import requests

   def naive_retry(n=5):
       for i in range(n):
           r = requests.post("http://localhost:6000/charge", json={})
           if r.status_code == 200:
               return r.json()
       raise Exception("failed after retries")
   ```
3. Fire 20 concurrent "customers" hitting `naive_retry()` at once (use `concurrent.futures.ThreadPoolExecutor`). Watch the Flask server's request log — count how many total requests hit `/charge` in the burst.
4. Now rewrite the client with Tenacity's exponential backoff + full jitter:
   ```python
   from tenacity import retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception_type

   @retry(stop=stop_after_attempt(5), wait=wait_exponential_jitter(initial=1, max=10))
   def smart_retry():
       r = requests.post("http://localhost:6000/charge", json={})
       if r.status_code != 200:
           raise Exception("retriable failure")
       return r.json()
   ```
5. Re-run the same 20-concurrent-customer burst with `smart_retry()`. Compare total request count and how spread-out (in time) the requests were, versus the naive version.

**Success criteria:** You observe, with real numbers, that the naive retry client generates a request-count spike concentrated in a short window, while the backoff+jitter version spreads the same retries over a longer, staggered window — and can explain why that difference matters for a real struggling dependency's ability to recover.

---

### Lab 4 — Add an idempotency key and prove it prevents double-processing

1. Modify `flaky_service.py` to track idempotency keys in memory:
   ```python
   seen_keys = {}

   @app.route("/charge", methods=["POST"])
   def charge():
       key = request.headers.get("Idempotency-Key")
       if key and key in seen_keys:
           return seen_keys[key]   # return the ORIGINAL result, don't reprocess
       if random.random() < FAIL_RATE:
           time.sleep(2)
           return {"error": "timeout"}, 503
       result = ({"status": "charged", "charge_id": str(uuid.uuid4())}, 200)
       if key:
           seen_keys[key] = result
       return result
   ```
2. Modify the Tenacity-based client to generate ONE idempotency key before the retry loop and send it with every attempt (not regenerated per attempt).
3. Add a counter to the server tracking how many times a *new* charge was actually created (not a cache hit) per idempotency key, and log it.
4. Run several retried calls (force `FAIL_RATE` high enough to guarantee at least one retry per call) and confirm the server's log shows exactly one "new charge created" per idempotency key, even when the client made multiple attempts.

**Success criteria:** You can show, from server-side logs, that N retries of the same logical request under one idempotency key produced exactly one actual charge — proving the mechanism prevents duplicate side effects.

---

### Cleanup

```bash
pkill -f flaky_service.py
```

### Stretch challenge

Add a simple circuit breaker (using `pybreaker` or hand-rolled) in front of the Tenacity-wrapped client call, tripping after 5 consecutive failures with a 15-second cooldown. Set `FAIL_RATE = 1.0` (always fails) and confirm that after the breaker trips, subsequent calls fail immediately (sub-millisecond) instead of going through the full retry-with-backoff sequence — demonstrating that the breaker takes precedence over continued retrying against a known-dead dependency.
