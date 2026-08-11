# Postmortem: Supervisor Health-Check Killing Live Workers (July 2026)

**System:** Pnlytics (uvicorn workers on a Windows VPS, external uptime monitoring)
**Impact:** ~8.9% availability flapping on the public endpoint; long-running data imports dying mid-run. No data corruption (imports are idempotent and resumable).
**Status:** Resolved — one-line configuration fix after evidence-first diagnosis.

## Symptom

The uptime monitor showed the site flapping — short, frequent windows of failed checks totalling ~8.9% — and background import jobs were dying partway through with no error in application logs. Workers were exiting with Windows status `0x10000` roughly 70 times per day.

## Investigation

The tempting explanation was resource contention from a co-tenant application on the same VPS. Instead of acting on that hunch, I instrumented process exits and captured 9 out of 9 worker deaths with full context. All nine correlated with the same trigger: uvicorn's **worker health-check timeout, default 5 seconds**. During CPU-contended periods (US market hours, when webhook and import load peaks), a busy worker took longer than 5 s to answer its supervisor's health probe — so the supervisor `TerminateProcess`-ed a perfectly healthy worker mid-request. The monitor flap and the dying imports were the same bug wearing two costumes.

The co-tenant application was formally cleared — worth stating, because it had been the leading suspect.

## Fix

`--timeout-worker-healthcheck 30` on the daemon start command. Flapping stopped; import kills stopped. Slow-import behavior during market hours remained (genuine contention, tracked separately) — the fix removed the *kills*, not the load.

## Lessons

- Supervisor defaults encode someone else's load assumptions. A 5 s liveness deadline is an SLO you didn't agree to.
- Correlate uptime flaps with **process exit codes**, not vibes — nine captured deaths beat any theory.
- Clear the innocent suspect explicitly. "It's the noisy neighbor" is the ops equivalent of blaming the intern.
