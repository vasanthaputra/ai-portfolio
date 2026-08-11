# Postmortem: Database Egress Storm (July 2026)

**System:** Pnlytics (FastAPI + Supabase Postgres, multi-tenant SaaS)
**Impact:** Database egress overage for the month; no data loss, no user-visible outage.
**Status:** Resolved — fixes shipped same day as root-cause confirmation.

## Symptom

Monthly database egress came in far above what the product's traffic could plausibly explain. Instrumenting egress by endpoint showed ~98% of it came from one pattern: full fetches of the processed-trades table (~24 MB per call), repeated hundreds of times. The month's total included ~743 runs of the trade-reprocessing pipeline — an order of magnitude more than legitimate triggers could account for.

## Root cause

The reprocessing pipeline was guarded by a distributed lock with a short TTL (300 seconds) and **no fencing**. Reprocess runs regularly exceeded the TTL under load, so the lock expired while the first worker was still running; a second worker then acquired the "free" lock and started the same job. Each run refetched the full processed dataset. Stale locks plus retries compounded into reprocess storms — a classic check-then-act race, exactly the failure mode fencing tokens exist to prevent.

## Fix (canonical pattern: TTL lease + fencing token)

1. **Fencing tokens** — a monotonic generation counter added to the lock (per Kleppmann's *How to do distributed locking*). Every write path in the pipeline must present the current generation; a mismatch raises a `StaleLockError` instead of writing.
2. **TTL raised 300 s → 6 h** to match the real worst-case job duration, so leases stop expiring mid-run.
3. **Cached-return drains** — concurrent requests for the same result now wait on the in-flight computation instead of triggering their own full fetch.
4. **Standing rule:** any new write inside the lock's critical section must pass the generation check — enforced in review.

## Lessons

- A TTL lock without a fencing token is not a lock; it's a race with a timer.
- Egress is an operations *and* security signal — meter it per endpoint before you need to.
- When a resource bill spikes, treat it like an incident: instrument first, hypothesize second.
