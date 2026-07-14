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

### Open Questions and Considerations

| # | Concern | Detail |
|---|---------|--------|
| 1 | State machine bypass | Drops the `DISPATCHED` precondition. "Upsert" is a fundamentally different model from the existing state machine. Requires a new callback Lambda code path that coexists with the existing PR/push state machine. |
| 2 | Trust model change | The relay can verify the SHA is real, but not that CI actually ran against it. A downstream could self-report results for a SHA it never tested. Is OIDC + SHA-existence sufficient, or do we need execution attestation? |
| 3 | No SHA alignment | Each downstream independently fetches HEAD. If `main` advances between repos' crons, they test different commits. Cross-backend comparison on HUD may be fragmented. |
| 4 | Missing-run detection is silent | No `DISPATCHED` record means no zombie sweeper coverage. If a downstream's cron silently breaks, nobody on the relay side knows. Should we add a "last seen" heartbeat per repo? |
| 5 | SHA overwrite on re-runs | Upsert keyed by `(repo, SHA)` overwrites previous results. No audit trail of multiple runs against the same SHA. Should we maintain run history? |
| 6 | No timing metrics | Without `dispatched_at`, queue time metrics are lost. |
| 7 | SHA validation adds dependency | `GET /repos/pytorch/pytorch/commits/{sha}` requires GitHub API availability and rate limits. Caching strategy needs to be defined. |
| 8 | Callback payload contract undefined | Current callbacks carry `delivery_id`, PR metadata. A nightly self-report has different fields. The new payload schema needs to be specified. |
| 9 | Callback Lambda complexity | New callback Lambda code path (~200 LOC), SHA validation + caching, upsert logic. The complexity shifts from AWS resources to Lambda code. |

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

## Drawbacks

- Introduces a second callback model (upsert) alongside the existing state machine, increasing Lambda complexity.
- Nightly failures have no upstream PR to annotate — requires a separate notification mechanism.
- Concurrent nightly and PR-triggered runs may compete for downstream CI resources.
- No central visibility into whether downstream nightlies are running or silently broken.
- SHA fragmentation across backends makes cross-repo nightly comparison harder.

## Alternatives

**Do nothing.** Downstream repos run nightly CI independently and report results in their own dashboards. PyTorch maintainers have no visibility into nightly downstream health. This is the current state.

**Central dispatch (EventBridge or upstream workflow).** The relay centrally triggers nightly dispatches to downstream repos, preserving the full state machine. This guarantees SHA alignment, zombie detection, and timing metrics but requires either new AWS infrastructure or changes to `pytorch/pytorch`.

## Prior Art

- **GitHub Actions scheduled workflows**: Widely used for nightly builds across the PyTorch ecosystem (e.g., `pytorch/pytorch` nightly builds, `pytorch/vision` nightly tests).
- **RFC-0050**: The original CRCR RFC that established the dispatch → callback → HUD pipeline for `pull_request` events.
- **RFC-0054**: HUD integration RFC that defined the ClickHouse schema and dashboard views for CRCR results.

## Unresolved Questions

### For WG Discussion

1. **State machine divergence:** Is the WG comfortable maintaining two distinct callback models — state machine (`DISPATCHED → IN_PROGRESS → COMPLETED`) for PR/push and upsert for nightly/periodic — in the same Lambda?

2. **Trust model:** Without a `DISPATCHED` record, how do we verify that a downstream repo actually ran CI against the SHA it claims? Is OIDC + SHA-existence sufficient?

3. **SHA alignment:** If downstream repos fetch `main` HEAD independently, they will test different SHAs when `main` advances between crons. Is fragmented cross-backend comparison on HUD acceptable?

4. **Missing-run detection:** Without `DISPATCHED` records, the zombie sweeper cannot detect silent cron failures in downstream repos. What replaces this?

5. **Scope:** Should all allowlisted repos be eligible for nightly self-report, or only repos at a certain level (e.g., L3+)?

6. **SHA policy:** Always `main` HEAD, or allow per-repo target branches (nightly branch, viable-strict, release branches)?

7. **Failure SLA:** Is a HUD view sufficient, or do we need active notifications (Slack, issues)?

8. **Periodic vs. nightly:** Do we need both `nightly` and `periodic` event types from day one, or start with `nightly` only and add `periodic` later?

### Shared Considerations

**HUD Changes.** The HUD currently groups results by `pr_number`. For nightly runs there is no PR. Use `pr_number = 0` as a sentinel for non-PR runs. The `event_type` field already flows through the pipeline — it just needs to carry `nightly` or `periodic` instead of `pull_request`. Add a filter/view on `hud.pytorch.org/crcr` for non-PR results, or a dedicated `/crcr/nightly` page.

**Allowlist Scoping.** Not all L1+ repos should report nightly results. Consider:

```yaml
L2:
  - org/repo:
      oncalls: user1, user2
      nightly: true    # opt-in to nightly self-report
```

Or gate on allowlist level (e.g., only L3+ can self-report nightlies by default).

**SHA Selection.** For nightly runs, which SHA to test against?
- `main` HEAD at cron time (most common, default)
- Latest release tag (for release compatibility testing)
- A specific branch (e.g., `release/2.x`)

**Failure Notifications.** PR failures are visible on the PR. Nightly failures have no PR to annotate.
- Dedicated Slack channel for CRCR nightly failures
- Auto-create GitHub issues on consecutive failures
- HUD dashboard alert on `/crcr/nightly`
- Email to repo oncalls from the allowlist

**Deduplication.** If a nightly self-report arrives while a PR-triggered run is in progress for the same downstream repo, they are handled independently (different `delivery_id` / event type). However, concurrent builds may compete for downstream CI resources. Consider whether the downstream workflow should use `concurrency:` groups to avoid parallel nightly + PR builds.

## Resolution

TBD — pending WG discussion.

### Level of Support

TBD

### Next Steps

TBD

#### Tracking Issue

TBD
