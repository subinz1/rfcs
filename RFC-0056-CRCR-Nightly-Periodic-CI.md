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

Instead of a central scheduler dispatching *to* downstream repos, each downstream repo drives its own schedule and reports results back. The relay becomes a **validating ingest endpoint** for nightly/periodic events. The full state machine is replaced with a **single-callback model**:

| | PR / push (existing) | Nightly / periodic (new) |
|---|---|---|
| **Trigger** | Upstream webhook → relay dispatches to downstream | Downstream cron (self-triggered) |
| **State machine** | `DISPATCHED → IN_PROGRESS → COMPLETED` (two callbacks, Redis state tracking) | No state machine — single callback with final result |
| **Callbacks** | Two: `in_progress` then `completed` | One: `completed` only |
| **Redis** | Required (state tracking + zombie sweeper) | Not used |
| **Validation** | GitHub webhook signature (`X-Hub-Signature-256`) | OIDC token + SHA existence |

This significantly simplifies the relay path for nightly/periodic: no Redis writes, no state transitions, no zombie sweeper coverage. The downstream workflow runs to completion and reports the final result in a single callback.

### SHA Sources

| Event type | Branch | SHA source | Rationale |
|------------|--------|------------|-----------|
| `nightly` | [`pytorch/pytorch/tree/nightly`](https://github.com/pytorch/pytorch/tree/nightly) | Top-of-tree commit on the `nightly` branch | The `nightly` branch is updated daily by [`trigger_nightly_core.yml`](https://github.com/pytorch/test-infra/blob/main/.github/workflows/trigger_nightly_core.yml). It represents the latest nightly-validated state of PyTorch. |
| `periodic` | `main` or `viable/strict` | Top-of-tree commit on the target branch | Periodic tests run against the latest `main` HEAD or the latest viable/strict commit. |

### Flow

```
Downstream repo's cron schedule (e.g., daily 02:00 UTC)
    ↓
Fetch top-of-tree SHA:
    - Nightly:  git ls-remote pytorch/pytorch refs/heads/nightly
    - Periodic: git ls-remote pytorch/pytorch refs/heads/main
    ↓
Runs CI against that SHA (build, test, etc.)
    ↓
Single callback to the relay (no in_progress step):
    - OIDC token (proves repo identity)
    - dispatch_id = the commit SHA (idempotent, correlatable,
      maps directly to github.com/pytorch/pytorch/commit/<sha>)
    - event_type = "nightly" or "periodic"
    - status = "completed"
    - conclusion = "success" | "failure" | "timed_out"
    ↓
Relay validates:
    1. OIDC token → repo is on allowlist with nightly enabled
    2. GET /repos/pytorch/pytorch/commits/{sha} → SHA exists
    ↓
Direct upsert to DynamoDB (no Redis, no state machine)
    ↓
DynamoDB → ClickHouse replicator → HUD
```

### Example: Downstream Nightly Workflow

```yaml
# downstream-repo/.github/workflows/crcr-nightly.yml
name: CRCR Nightly CI
on:
  schedule:
    - cron: '0 2 * * *'  # daily at 02:00 UTC
  workflow_dispatch: {}   # manual re-trigger

jobs:
  nightly:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: Get nightly branch HEAD SHA
        id: sha
        run: |
          SHA=$(git ls-remote https://github.com/pytorch/pytorch refs/heads/nightly | cut -f1)
          echo "sha=$SHA" >> "$GITHUB_OUTPUT"
          echo "Testing against nightly SHA: $SHA"

      - name: Build and test against nightly
        run: |
          # Clone pytorch at the nightly SHA, build, run tests
          ...

      # Single callback — no in_progress step needed for nightly.
      # Reports the final result directly to the relay.
      - name: Report results to CRCR
        if: always()
        uses: ./.github/actions/cross-repo-ci-relay
        with:
          dispatch_id: ${{ steps.sha.outputs.sha }}
          event_type: nightly
          status: completed
          conclusion: ${{ job.status }}
```

### Advantages

| # | Advantage | Detail |
|---|-----------|--------|
| 1 | No trigger from pytorch/pytorch | Nothing in upstream emits an event. No new workflows, branches, or tags. |
| 2 | No new AWS infrastructure | No EventBridge, no Terraform. |
| 3 | Self-service schedule | Each downstream repo owns its cron. Changes are PRs to the downstream repo. |
| 4 | Manual re-trigger | `workflow_dispatch` on the downstream cron workflow re-runs the nightly. |
| 5 | Coordination-free correlation | `dispatch_id` is the nightly branch HEAD SHA — meaningful, idempotent, and lets HUD map runs directly to `github.com/pytorch/pytorch/commit/<sha>`. |
| 6 | Leverages existing nightly branch | The `nightly` branch already exists and is updated daily by `trigger_nightly_core.yml`. No new infrastructure needed to determine the SHA. |

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

## Replay & Recovery

Nightly/periodic pipelines are **idempotent by design**: the `delivery_id` is the upstream commit SHA, and the callback upserts into DynamoDB, so re-running the same workflow for the same SHA is safe and produces no duplicates.

**Manual replay procedure** (Option 1 — adopted for initial launch):

1. Navigate to the downstream repo's Actions tab (e.g., `TorchedHat/pytorch-redhat-ci` → Actions → "CRCR Nightly").
2. Click "Run workflow" (`workflow_dispatch` trigger is already enabled).
3. The workflow fetches the current `nightly` branch HEAD SHA, runs CI, and reports results via the callback action — identical to a cron-triggered run.

No centralized replay endpoint is needed at this stage. Automated missed-nightly detection (self-healing re-triggers) may be considered in a future iteration based on WG feedback.

## Multi-CI Provider Authentication

The self-report model currently relies on GitHub Actions OIDC tokens for caller identity. To support downstream backends that run CI on other platforms (e.g., Buildkite, GitLab CI), the relay's `jwt_helper` needs to become multi-issuer.

### Problem

`jwt_helper.verify_oidc_token()` is hardcoded to validate tokens from a single issuer (`https://token.actions.githubusercontent.com`). Backends running on Buildkite (e.g., vLLM at `vllm-project/vllm`) cannot authenticate to the callback Lambda because their OIDC tokens come from a different issuer (`https://agent.buildkite.com`).

### Design: Issuer-Based Dispatch

Add a registry of trusted issuers, each with its own JWKS endpoint and claim-extraction logic. The `verify_oidc_token` function becomes:

1. Strip `Bearer ` prefix (existing)
2. Decode the JWT header **unverified** to read the `iss` claim
3. Look up the issuer in the registry → reject unknown issuers with 401
4. Fetch the signing key from the issuer-specific JWKS endpoint
5. Verify the signature, audience (`pytorch-cross-repo-ci-relay`), and issuer
6. Extract `verified_repo` using the issuer-specific claim mapper

```python
_ISSUERS = {
    "https://token.actions.githubusercontent.com": {
        "jwks": "https://token.actions.githubusercontent.com/.well-known/jwks",
        "repo_claim": lambda claims: claims["repository"],
    },
    "https://agent.buildkite.com": {
        "jwks": "https://agent.buildkite.com/.well-known/jwks",
        "repo_claim": lambda claims: _buildkite_to_repo(
            claims["organization_slug"], claims["pipeline_slug"]
        ),
    },
}
```

### Buildkite Identity Mapping

Buildkite OIDC tokens have no `repository` claim. They provide `organization_slug` and `pipeline_slug`. A static mapping resolves these to a GitHub `owner/repo`:

```python
_BUILDKITE_REPO_MAP = {
    ("vllm", "vllm-ci"): "vllm-project/vllm",
    # Add more as backends onboard
}
```

The mapping is maintained in `jwt_helper.py` so that identity resolution is complete before the callback handler runs. Everything downstream of `verified_repo` (callback handler, allowlist, Redis, HUD) is unchanged.

### CI Provider Reference

| CI Engine | Issuer (`iss`) | JWKS Endpoint | Repo Claim |
|-----------|---------------|---------------|------------|
| GitHub Actions | `https://token.actions.githubusercontent.com` | `.../.well-known/jwks` | `claims["repository"]` |
| Buildkite | `https://agent.buildkite.com` | `.../.well-known/jwks` | `(organization_slug, pipeline_slug)` → static map |
| GitLab CI | `https://gitlab.com` | `.../-/oauth/discovery/keys` | Future — `claims["project_path"]` |
| CircleCI | `https://oidc.circleci.com/org/<ID>` | `.../.well-known/jwks.json` | Future — org-specific mapping |

Only GitHub Actions and Buildkite are in scope for initial implementation. GitLab and CircleCI can be added later by extending the `_ISSUERS` registry.

### Implementation Scope

| Component | Change | LOC |
|-----------|--------|-----|
| `utils/jwt_helper.py` | Multi-issuer dispatch + Buildkite mapping | ~45 |
| `callback/lambda_function.py` | None — interface unchanged | 0 |
| `callback/callback_handler.py` | None | 0 |
| Tests | 3 new test cases (Buildkite verify, unknown pipeline, unknown issuer) | ~15 |
| **Total** | | **~60** |

Tracking issue: [pytorch/test-infra#8326](https://github.com/pytorch/test-infra/issues/8326)

### File Changes

| File | Change |
|------|--------|
| `callback/lambda_function.py` | New code path: detect `event_type ∈ {nightly, periodic}`, skip state machine entirely (no Redis), validate OIDC + SHA, single upsert to DynamoDB |
| `callback/sha_validator.py` | New module: `GET /repos/pytorch/pytorch/commits/{sha}` with TTL cache to avoid repeated GitHub API calls |
| `allowlist.yml` | Add `nightly: true/false` per-repo flag to control which repos can self-report nightly results |
| Downstream workflow (per repo) | New `schedule: cron` workflow: fetch top-of-tree SHA from `nightly` branch (or `main` for periodic), run CI, call callback with `event_type: nightly` and `dispatch_id: <SHA>` |
| HUD (`torchci/`) | Filter/view for `event_type != pull_request` on `/crcr/nightly` or dedicated nightly page |

## Previously Considered Options

Two alternative approaches were evaluated before arriving at the authenticated self-report design:

**Option A: EventBridge Cron → Webhook Lambda.** An AWS EventBridge rule on a cron schedule invokes the webhook Lambda directly. The Lambda fetches `pytorch/pytorch` main HEAD SHA, builds a synthetic `client_payload`, and dispatches to downstream repos via the existing `_dispatch_to_allowlist()` path. This preserves the full state machine and guarantees SHA alignment across all backends. However, it introduces new AWS infrastructure (EventBridge rule, Terraform config, CloudWatch alarms) and centralizes schedule control — downstream repos cannot customize their own cron timing without additional EventBridge rules.

**Option B: Upstream Cron Workflow in pytorch/pytorch → Webhook Lambda.** A `schedule: cron` workflow in `pytorch/pytorch` constructs a synthetic payload and POSTs it to the webhook Lambda endpoint with OIDC authentication. This gives upstream visibility (schedule appears in the Actions tab) and built-in manual re-trigger via `workflow_dispatch`. However, it requires adding a second authentication path (OIDC or shared secret) to the webhook Lambda, changes to `pytorch/pytorch` requiring maintainer approval, and depends on GitHub cron reliability.

Both options were set aside in favor of the self-report model because they require either new AWS infrastructure or upstream repo changes, while the proposed design keeps all changes within the callback Lambda and downstream repos.

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
