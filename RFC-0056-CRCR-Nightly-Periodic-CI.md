# CRCR Support for Nightly & Periodic CI

**Authors:**
* @groenenboomj
* @jewelkm89
* @subinz1

**Status:** Draft — for CRCR Working Group discussion

**Date:** June 2026

## Summary

Extend the Cross-Repository CI Relay (CRCR) to support nightly and periodic CI schedules for downstream repositories. Currently, CRCR only dispatches on `pull_request` and `push` events from `pytorch/pytorch`. This RFC proposes adding an authenticated self-report model so downstream backends can independently schedule nightly/periodic CI and report results to the PyTorch HUD.

## Motivation

CRCR currently dispatches downstream CI on `pull_request` and `push` events from `pytorch/pytorch`. The webhook Lambda receives these GitHub webhook events, generates a `delivery_id`, sends `repository_dispatch` to all allowlisted downstream repos, and sets `DISPATCHED` in Redis. Downstream repos run their CI, then report results back via the callback Lambda, which validates the state machine (`DISPATCHED → IN_PROGRESS → COMPLETED`) and forwards metrics to HUD.

Nightly and periodic runs have no upstream trigger. They are cron-scheduled jobs (e.g., nightly builds against `main` HEAD, weekly compatibility tests against release branches). This creates two blockers:

1. **No dispatch.** Without an upstream webhook event, there is no `repository_dispatch` to downstream repos. Downstream nightly jobs would have to self-trigger via their own `schedule: cron`.

2. **No callback path.** The state machine rejects callbacks without a prior `DISPATCHED` record (HTTP 400: "no prior dispatch"). Even if a downstream repo runs a nightly job and tries to report results, the callback is rejected.

**Impact:** Downstream backends cannot report nightly/periodic CI results to HUD. This is a gap for L3/L4 backends that need to show nightly compatibility on `hud.pytorch.org/crcr`.

## Current Architecture

### Supported Events

The webhook Lambda accepts two GitHub event types (see `_SUPPORTED_EVENTS` in `webhook/lambda_function.py`):

| Event | Source | When |
|-------|--------|------|
| `pull_request` | GitHub webhook | PR opened, reopened, synchronize, closed |
| `push` | GitHub webhook | Push to any branch in `pytorch/pytorch` |

Both follow the same dispatch path:

```
GitHub webhook (pull_request or push)
    ↓
Webhook Lambda:
    1. Verify GitHub signature (X-Hub-Signature-256)
    2. Check event type ∈ {pull_request, push}
    3. Check repo == upstream_repo (pytorch/pytorch)
    4. Generate delivery_id from X-GitHub-Delivery header
    5. For each allowlisted downstream repo:
       - Mint GitHub App installation token
       - Send repository_dispatch(event_type, client_payload)
       - Set DISPATCHED state in Redis
    ↓
Downstream repo receives repository_dispatch
    ↓
Downstream workflow runs CI, calls callback action:
    - in_progress → Callback Lambda validates state, sets IN_PROGRESS in Redis
    - completed   → Callback Lambda validates state, sets COMPLETED in Redis
    ↓
Callback Lambda forwards trusted + untrusted payloads to HUD
    ↓
HUD → DynamoDB → ClickHouse → hud.pytorch.org/crcr
```

### Downstream Workflow Structure

Downstream repos listen for `repository_dispatch` with the event types they want:

```yaml
# Existing downstream pattern
on:
  repository_dispatch:
    types: [pull_request, push]  # L1 receiver already handles both
```

The `client_payload` always contains `event_type`, `delivery_id`, and the upstream webhook payload. Downstream workflows branch on `event_type` to extract PR number, SHA, ref, etc.

## Proposed Design: Authenticated Self-Report

*Proposed by @atalman in [PR #98 comment](https://github.com/pytorch/rfcs/pull/98#issuecomment-4962790260).*

Instead of a central scheduler dispatching *to* downstream repos, each downstream repo drives its own schedule and reports results back. The relay stops being a trigger and becomes a **validating ingest endpoint**. The `DISPATCHED → IN_PROGRESS → COMPLETED` state machine precondition is dropped for nightly/periodic events and replaced with authorization + SHA-validity at callback time.

```
Downstream repo's own cron schedule
    ↓
Fetches pytorch/pytorch HEAD SHA (or nightly/viable-strict ref)
    ↓
Runs CI against that SHA
    ↓
Calls back to the relay with:
    - OIDC token (proves repo identity)
    - dispatch_id = pytorch/pytorch commit SHA
    - event_type = "nightly" or "periodic"
    ↓
Relay validates:
    1. OIDC token → repo is on allowlist
    2. GET /repos/pytorch/pytorch/commits/{sha} → SHA is real
    ↓
Upsert record keyed by (repo, SHA) → HUD
```

### Advantages

| # | Advantage | Detail |
|---|-----------|--------|
| 1 | No trigger from pytorch/pytorch | Nothing in upstream emits an event. No new workflows, branches, or tags. |
| 2 | No new AWS infrastructure | No EventBridge, no Terraform. |
| 3 | Self-service schedule | Each downstream repo owns its cron. Changes are PRs to the downstream repo. |
| 4 | Manual re-trigger | `workflow_dispatch` on the downstream cron workflow re-runs the nightly. |
| 5 | Coordination-free correlation | `dispatch_id` is the SHA, derivable by any actor independently. |

### Implementation Effort

| Component | Work | Effort |
|-----------|------|--------|
| Callback Lambda upsert path | New code path: skip state machine, validate OIDC + SHA, upsert | ~2 days |
| SHA validation + caching | GitHub API integration + cache layer | ~1 day |
| Downstream cron workflow | New workflow in each downstream: fetch SHA, run CI, call callback | ~1 day per repo |
| HUD view | Filter/view for non-PR results grouped by SHA | ~1 day |
| Testing | End-to-end test with `TorchedHat/pytorch-redhat-ci` | ~1 day |
| **Total** | | **~5-6 days** |

## Metrics

- **Nightly callback completion rate**: Percentage of expected nightly runs (per downstream cron schedule) that successfully report results via callback.
- **HUD coverage**: Number of downstream backends with nightly results visible on `hud.pytorch.org/crcr`.
- **Time-to-detection**: How quickly a nightly regression in a downstream backend is surfaced on HUD.

### File Changes

| File | Change |
|------|--------|
| `callback/lambda_function.py` | New code path: detect `event_type ∈ {nightly, periodic}`, skip state machine, validate OIDC + SHA, upsert record to DynamoDB |
| `callback/sha_validator.py` | New module: `GET /repos/pytorch/pytorch/commits/{sha}` with TTL cache to avoid repeated GitHub API calls |
| `allowlist.yml` | Add `nightly: true/false` per-repo flag to control which repos can self-report nightly results |
| Downstream workflow (per repo) | New `schedule: cron` workflow that fetches `pytorch/pytorch` HEAD SHA, runs CI, and calls the callback with `event_type: nightly` and `dispatch_id: <SHA>` |
| HUD (`torchci/`) | Filter/view for `event_type != pull_request` on `/crcr/nightly` or dedicated nightly page |

## Prior Art

- **GitHub Actions scheduled workflows**: Widely used for nightly builds across the PyTorch ecosystem (e.g., `pytorch/pytorch` nightly builds, `pytorch/vision` nightly tests).
- **RFC-0050**: The original CRCR RFC that established the dispatch → callback → HUD pipeline for `pull_request` events.
- **RFC-0054**: HUD integration RFC that defined the ClickHouse schema and dashboard views for CRCR results.

## Resolution

TBD — pending WG discussion.

### Level of Support

TBD

### Next Steps

TBD

#### Tracking Issue

TBD
