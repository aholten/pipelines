# Safe Dependabot auto-merge plan for `kubeflow/pipelines`

**Prepared:** 2026-08-08  
**Upstream baseline:** `kubeflow/pipelines@1630f8063b3434217f54da67aea221b910ea238b` (`master`)  
**Decision status:** **Design approved for staged validation; production enablement is blocked on administrator and Tide/Prow decisions.**  
**Mutation boundary:** this report made no changes to `kubeflow/pipelines`, `aholten/pipelines`, repository settings, branches, pull requests, or workflows.

## Executive recommendation

Do **not** implement “merge every Dependabot PR when the visible CI looks green.” Instead:

1. Keep GitHub/Tide policy—not a workflow—as the merge authority.
2. Add one metadata-only, no-checkout eligibility workflow that can only request **GitHub native squash auto-merge**. It must never call an immediate merge, approve the PR, change branch rules, create a successful status, or bypass a gate.
3. Start with **Go modules only, direct production dependencies, SemVer patch only, ungrouped, Dependabot-authored and Dependabot-only commits, and only `go.mod`/`go.sum` in the five configured Go module directories**. Keep Docker, minor/major, grouped, indirect/development, executable, workflow, deployment, and generated-file changes human-reviewed.
4. Do not turn on the write step until an administrator exports the effective `master` policy and Tide owners choose one merger model. Today the public rules prove DCO and squash-only PR merging, but the integration cannot see effective legacy protection, Actions token settings, hidden bypasses, auto-merge, or merge-queue settings.
5. Use the built-in `GITHUB_TOKEN` only if a synchronized fork PoC proves that the current repository policy honors the explicit `contents: write` and `pull-requests: write` permissions shown in GitHub’s current Dependabot auto-merge tutorial. GitHub’s own current pages conflict with the current `dependabot/fetch-metadata` README on this point; do not resolve that conflict by assumption. If the built-in token is denied, use a dedicated, repository-only GitHub App installation token—not a classic PAT—and grant no administration or bypass capability.
6. Keep merge queue out of the first rollout. No checked-in workflow currently declares `merge_group`; GitHub states required Actions checks will not run for a queue unless those workflows handle `merge_group`.

**Recommendation to Jeff Spahr:** approve a report-only proof and policy audit now; approve live native auto-merge later only for the narrow Go patch class after the fork test, Tide decision, and branch-policy export pass. This answers the operational request without treating “all CI passed” as sufficient authorization.

## Evidence and scope

The repository was freshly cloned read-only and rechecked at the baseline SHA. Immutable source links:

* Dependabot configuration: [`.github/dependabot.yml`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/.github/dependabot.yml)
* CI aggregate: [`.github/workflows/ci-checks.yml`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/.github/workflows/ci-checks.yml)
* exact-head `ci-passed` writer: [`.github/workflows/add-ci-passed-label.yml`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/.github/workflows/add-ci-passed-label.yml)
* workflow-run approval: [`.github/workflows/gh-workflow-approve.yml`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/.github/workflows/gh-workflow-approve.yml)
* contributor policy: [`CONTRIBUTING.md`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/CONTRIBUTING.md)
* Prow ownership: [`OWNERS`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/OWNERS) and [`.github/OWNERS`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/.github/OWNERS)

Read-only GitHub API evidence was rechecked through:

* `GET /repos/kubeflow/pipelines`
* `GET /repos/kubeflow/pipelines/git/ref/heads/master`
* `GET /repos/kubeflow/pipelines/rulesets/{14287837,14287485,14287367,11135155}`
* `GET /repos/kubeflow/pipelines/branches/master/protection` (**403**)
* `GET /repos/kubeflow/pipelines/actions/permissions` (**403**)
* `GET /repos/kubeflow/pipelines/actions/permissions/workflow` (**403**)
* `GET /repos/kubeflow/pipelines/pulls/13921` and its head check runs

The public repository tree contains no `CODEOWNERS`. Prow-style `OWNERS` files are not a GitHub CODEOWNERS branch-rule gate.

## Verified current-state PR/check/approval/merge gate map

| Order | Gate | Verified behavior | Enforcement confidence |
|---:|---|---|---|
| 1 | PR identity and target | Dependabot PR author is `dependabot[bot]`; target is `master`. | API/source observable. |
| 2 | PR route and merge method | Active org ruleset `no-push-main` (ID `14287485`) requires a pull request and allows only `squash`. It visibly requires 0 approvals, no CODEOWNER review, no last-push approval, no stale-review dismissal, and no conversation resolution. | Hard visible rule; hidden legacy rules may add gates. |
| 3 | DCO | Active org ruleset `enforce-dco` (ID `14287837`) requires context `DCO` from integration ID `1861`; `strict_required_status_checks_policy` is false. | Hard visible rule. |
| 4 | CI authorization | `.github/workflows/gh-workflow-approve.yml` uses `pull_request_target`. Non-member/non-collaborator PRs need `ok-to-test` before it approves `action_required` workflow runs. | Verified source behavior; the operator’s `aholten/oss-test-infra` lead may explain Prow-side auto-triggering and is under separate read-only audit (`t_db33e4da`). CI triggering/approval must remain separate from merge authorization. |
| 5 | CI aggregate eligibility | `.github/workflows/ci-checks.yml` requires `ok-to-test`, absence of `needs-ok-to-test`, waits for `action_required` runs, then polls all checks through pinned `wechuli/allcheckspassed@204ff63e…`, excluding only `Cleanup artifacts`, `Upload results`, `Agent`, and `Prepare`. | Verified source behavior. It is a dynamic aggregator, not a demonstrated protected required check. |
| 6 | Exact-head UI signal | `.github/workflows/add-ci-passed-label.yml` accepts only a successful aggregate artifact, re-queries the open PR, compares the recorded and current head SHA, rechecks labels, then writes `ci-passed`; it removes stale labels on relevant changes. | Good exact-head anti-TOCTOU behavior, but a mutable label is not branch protection. |
| 7 | Tide/Prow | Live PR #13921 exposes external `tide` status linking to `https://oss.gprow.dev/tide`. No production Tide submit policy is checked into this repo. | Material gate, but its live config/ownership is outside this source audit. |
| 8 | GitHub effective policy | GitHub finally evaluates all effective rules/protections and repository merge settings. | Incomplete visibility: protection and Actions settings endpoints return 403. |

### Live example, not a candidate

At assessment time [PR #13921](https://github.com/kubeflow/pipelines/pull/13921) was open, non-draft, authored by the verified Dependabot bot, and based on `master`, but was `behind`, had no auto-merge request, lacked `ok-to-test`/`ci-passed`, and had multiple failed API-server checks on head `ecc71c3b996b5340a3cb9147fff9833df9c4a923`. It demonstrates why author identity plus partial green CI is insufficient.

## Preferred least-privilege architecture

```text
pull_request event for master
  -> trusted workflow definition from base branch
  -> verify repository/base/open/draft/Dependabot App identity
  -> fetch signed Dependabot metadata (SHA-pinned action)
  -> reject grouped/non-Go/non-direct-production/non-patch/manual changes
  -> list changed files by API; accept exact Go manifest allow-list only
  -> emit structured report-only decision
  -> later: request GitHub native squash auto-merge
  -> GitHub effective rules + Tide ownership decision remain authoritative
```

Properties:

* `pull_request`, not `pull_request_target`, is preferred for the eligibility workflow.
* It never checks out the PR or downloads/executes a PR artifact.
* It does not run dependency code, `go` commands, repository scripts, or a PR-controlled action.
* Third-party actions are pinned to reviewed full SHAs. GitHub says a full commit SHA is the only immutable action reference [G4].
* Every activation is re-evaluated on `opened`, `reopened`, `synchronize`, `converted_to_draft`, and `ready_for_review`.
* A manually changed Dependabot PR is denied. `dependabot/fetch-metadata` verifies the author and commits by default; do not set `skip-verification` or `skip-commit-verification`.
* Native auto-merge waits for effective required checks and approvals. It is not an immediate merge [G1].
* The automation identity is never a ruleset bypass actor and has no `administration`, `actions`, `checks`, `statuses`, `issues`, `id-token`, packages, deployments, or environment permission.

### Important disputed token behavior

Two current first-party/official sources disagree:

* GitHub Docs currently says Dependabot-triggered `pull_request` workflows are read-only **by default**, but explicitly says the workflow `permissions` key can increase the token’s access; its current auto-merge example uses `pull_request`, `contents: write`, and `pull-requests: write` [G1, G2].
* The current `dependabot/fetch-metadata` README says Dependabot `pull_request` runs have a read-only token and its write examples therefore use `pull_request_target`.

The safe decision is empirical: retain `pull_request`, run a synchronized fork PoC, and observe the actual token. Do not switch to `pull_request_target` merely to get write authority. If current platform/repository policy denies the built-in token, use a dedicated App token in the same no-checkout workflow, stored as a Dependabot secret, or use a separately reviewed `workflow_run` control plane. The upstream setting remains unknown because the Actions endpoints returned 403.

## Exact proposed repository changes

### Change A — add a report-only workflow first

Add `.github/workflows/dependabot-auto-merge.yml` in a reviewed PR. This first version has read-only permissions and cannot merge:

```yaml
name: Dependabot auto-merge eligibility

on:
  pull_request:
    branches: [master]
    types: [opened, reopened, synchronize, converted_to_draft, ready_for_review]

permissions:
  contents: read
  pull-requests: read

concurrency:
  group: dependabot-automerge-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  classify:
    if: >-
      github.repository == 'kubeflow/pipelines' &&
      github.event.pull_request.user.login == 'dependabot[bot]' &&
      github.event.pull_request.user.type == 'Bot' &&
      github.event.pull_request.user.html_url == 'https://github.com/apps/dependabot' &&
      github.event.pull_request.base.ref == 'master' &&
      github.event.pull_request.state == 'open'
    runs-on: ubuntu-latest
    steps:
      - id: metadata
        uses: dependabot/fetch-metadata@d7267f607e9d3fb96fc2fbe83e0af444713e90b7
        with:
          github-token: ${{ github.token }}

      - name: Validate policy and changed files
        id: policy
        env:
          GH_TOKEN: ${{ github.token }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REPOSITORY: ${{ github.repository }}
          EVENT_ACTOR: ${{ github.actor }}
          IS_DRAFT: ${{ github.event.pull_request.draft }}
          DEPENDENCY_TYPE: ${{ steps.metadata.outputs.dependency-type }}
          UPDATE_TYPE: ${{ steps.metadata.outputs.update-type }}
          ECOSYSTEM: ${{ steps.metadata.outputs.package-ecosystem }}
          DEPENDENCY_GROUP: ${{ steps.metadata.outputs.dependency-group }}
          MAINTAINER_CHANGES: ${{ steps.metadata.outputs.maintainer-changes }}
        shell: bash
        run: |
          set -euo pipefail
          eligible=true
          reasons=()
          [[ "$EVENT_ACTOR" == 'dependabot[bot]' ]] || { eligible=false; reasons+=(actor); }
          [[ "$IS_DRAFT" == 'false' ]] || { eligible=false; reasons+=(draft); }
          [[ "$DEPENDENCY_TYPE" == 'direct:production' ]] || { eligible=false; reasons+=(dependency-type); }
          [[ "$UPDATE_TYPE" == 'version-update:semver-patch' ]] || { eligible=false; reasons+=(update-type); }
          [[ "$ECOSYSTEM" == 'gomod' ]] || { eligible=false; reasons+=(ecosystem); }
          [[ -z "$DEPENDENCY_GROUP" ]] || { eligible=false; reasons+=(grouped); }
          [[ "$MAINTAINER_CHANGES" == 'false' ]] || { eligible=false; reasons+=(maintainer-changes); }

          allowed='^(go\.(mod|sum)|api/go\.(mod|sum)|kubernetes_platform/go\.(mod|sum)|third_party/ml-metadata/go\.(mod|sum)|test/tools/project-cleaner/go\.(mod|sum))$'
          mapfile -t files < <(gh api --paginate \
            "repos/${REPOSITORY}/pulls/${PR_NUMBER}/files?per_page=100" \
            --jq '.[].filename')
          ((${#files[@]} > 0)) || { eligible=false; reasons+=(no-files); }
          for file in "${files[@]}"; do
            [[ "$file" =~ $allowed ]] || { eligible=false; reasons+=("path:$file"); }
          done

          printf 'eligible=%s\n' "$eligible" >> "$GITHUB_OUTPUT"
          printf '### Dependabot auto-merge eligibility\n\n' >> "$GITHUB_STEP_SUMMARY"
          printf -- '- head: `%s`\n- eligible: `%s`\n- reasons: `%s`\n' \
            "$GITHUB_SHA" "$eligible" "${reasons[*]:-none}" >> "$GITHUB_STEP_SUMMARY"
```

This copy-ready baseline deliberately contains no write permission and no live merge command. Its logs/summary are the dry-run evidence.

### Change B — live command after all gates pass

After report-only validation, change the permissions, insert a fail-closed stale-request reset **before** the metadata step, and append the guarded request step after policy evaluation:

```yaml
permissions:
  contents: write
  pull-requests: write

# insert as the first step, before fetch-metadata
      - name: Clear and verify any stale auto-merge request
        env:
          GH_TOKEN: ${{ github.token }}
          PR_URL: ${{ github.event.pull_request.html_url }}
        shell: bash
        run: |
          set -euo pipefail
          active="$(gh pr view "$PR_URL" --json autoMergeRequest \
            --jq '.autoMergeRequest != null')"
          if [[ "$active" == 'true' ]]; then
            gh pr merge --disable-auto "$PR_URL"
          fi
          [[ "$(gh pr view "$PR_URL" --json autoMergeRequest \
            --jq '.autoMergeRequest != null')" == 'false' ]]

# append after the policy step
      - name: Request native squash auto-merge
        if: steps.policy.outputs.eligible == 'true'
        env:
          GH_TOKEN: ${{ github.token }}
          PR_URL: ${{ github.event.pull_request.html_url }}
        run: gh pr merge --auto --squash "$PR_URL"
```

Reset-first matters: every synchronize/draft/readiness event removes a request evaluated against an older head before current metadata and paths can re-arm it. If disable/verification fails, the job fails and must page the owner; the fork test must prove the request was actually removed, including after a maintainer-authored synchronize event. GitHub’s current tutorial documents this permission shape and native command [G1]. Before upstream use, the fork must prove: (a) the token is actually write-capable; (b) the command creates a pending auto-merge request rather than an immediate/bypass merge; (c) reset-first reliably cancels an old request; and (d) a failing required check keeps it pending.

**Do not claim that Change B excludes security-update PRs.** The minimal metadata path does not reliably distinguish a scheduled version update from an alert-backed security update. If maintainers require “security updates stay manual,” fail closed until a dedicated repository-only App can perform `alert-lookup: true`; the current metadata action requires a PAT or App token for that lookup. Prefer an App with Dependabot-alert read access and no administration/bypass permission. A title, body, branch-name regex, or mutable label is not an acceptable substitute.

### Change C — effective `master` policy delta, owned by admins

Do not create a competing ruleset blind. Export all effective rules first. Apply the following **target properties** to the single effective policy through the administrator’s normal ruleset management process:

```yaml
# Desired state; map into the existing effective org/repo ruleset.
target: refs/heads/master
pull_request:
  allowed_merge_methods: [squash]
  required_approving_review_count: ADMIN_DECISION
  dismiss_stale_reviews_on_push: true
  require_last_push_approval: true
  required_review_thread_resolution: true
required_status_checks:
  strict_required_status_checks_policy: true
  checks:
    - context: DCO
      integration_id: 1861
    - context: ADMIN_VERIFIED_CI_AGGREGATE_CONTEXT
      integration_id: ADMIN_VERIFIED_SOURCE_APP_ID
bypass_actors: []
```

The CI aggregate context and App ID are placeholders because the assessment token cannot inspect effective protection and source does not prove which aggregate is required. An administrator must replace them from an export; do not copy the placeholder into GitHub.

Protect `.github/workflows/dependabot-auto-merge.yml`, `.github/workflows/ci-checks.yml`, `.github/workflows/add-ci-passed-label.yml`, `.github/workflows/gh-workflow-approve.yml`, and `.github/actions/**` with a GitHub `CODEOWNERS` rule or equivalent required-review policy. Prow `OWNERS` alone does not create that GitHub branch-rule protection.

### Change D — Dependabot configuration policy

No `.github/dependabot.yml` change is necessary for report-only mode. Its current Go groups intentionally combine minor/patch updates, so the initial `dependency-group == ''` policy will deny most grouped Go updates. That is an intentional low-volume safety start, not a bug.

If maintainers want a steady stream of individually testable patch candidates, split grouping explicitly in a separate reviewed PR:

```yaml
# Under each gomod update entry, replace broad minor/patch grouping policy only
# after maintainers accept the increased PR volume.
groups:
  golang-x:
    patterns: ["golang.org/x/*"]
    update-types: ["minor"]
  go-minor:
    patterns: ["*"]
    update-types: ["minor"]
# Patch updates then remain individual candidates.
```

Do not change Docker grouping or add ecosystems as part of the auto-merge PR.

## Dependency/update-type and ecosystem policy

| Class | Initial policy | Reason |
|---|---|---|
| Go, direct production, patch, ungrouped, only allowed `go.mod`/`go.sum` paths | Report-only, then eligible after evidence | Narrowest source-only change class already configured. |
| Go grouped patch/minor | Manual | Grouped updates widen blast radius; current config groups them deliberately. |
| Go minor or major | Manual | Compatibility/API risk and explicit major-review intent. |
| Go indirect or development | Manual | Metadata/diff can be broad or generated by solver changes. |
| Docker/image/manifests | Manual | Runtime and supply-chain impact; tags/digests do not map safely to SemVer policy. |
| Security update | Admin decision; manual unless an alert-backed classifier proves it is permitted | Minimal metadata does not reliably distinguish it. Do not infer from title/label. |
| npm/pip/GitHub Actions | Out of scope | Not configured for scheduled version updates at the baseline. |
| Any `.github/**`, source, scripts, generated code, deployment/release file | Deny | A merge-capable control plane must not accept executable or policy changes. |
| Maintainer-modified or non-Dependabot commit | Deny and cancel/review existing request | Prevents stale eligibility after human changes. |

## Security threat model and mitigations

| Threat | Failure mode | Required mitigation |
|---|---|---|
| Privileged PR execution | PR code steals a write token through `pull_request_target`, cache, artifact, or checkout. | Use `pull_request`; no checkout, artifacts, cache, dependency install, repository script, composite action, or PR-controlled code. |
| Actor spoofing | A branch/title/body resembles Dependabot. | Verify login, Bot type, App URL, repository/base, official metadata action, and its default commit verification. |
| Malicious dependency release | Legitimate Dependabot proposes compromised package code. | Direct-production patch-only start, exact path allow-list, all protected CI/security checks, human review for higher-risk classes, staged expansion. |
| Grouped or solver-expanded diff | A nominal patch changes many dependencies/files. | Require empty dependency group, inspect every changed filename, deny unknown paths, log dependency metadata. |
| TOCTOU/head movement | Eligibility was computed for an old head. | Per-PR concurrency cancellation, re-run on synchronize, use event/current PR head, Dependabot-only commit verification, native branch-rule re-evaluation; queue later for integrated-head proof. |
| Mutable `ci-passed`/check spoofing | A label or same-named status is mistaken for enforcement. | Never query the label as merge authority; require explicit strict checks bound to expected integration IDs where supported. |
| Stale human approval | Approval survives a new push. | Dismiss stale reviews and/or require last-push approval in effective policy; bot never approves. |
| Action supply-chain compromise | Mutable action tag steals write token. | Full-SHA pins, reviewed source/release, protect workflow files. |
| App/PAT theft | Credential can merge or administer broadly. | Prefer built-in token; otherwise one-repo App, short-lived installation token, minimum permissions, no admin/bypass; never classic PAT. |
| Two merger authorities | Tide and GitHub race or disagree. | Tide owners must designate one merger authority before live rollout. Do not bypass `tide` or emulate its policy. |
| Queue without CI | Queue SHA never receives required checks. | No queue until every required check handles `merge_group` and is tested. |
| Security-update misclassification | Alert-backed PR is treated as ordinary version update. | Make policy explicit; use App-backed alert lookup if exclusion is required; never infer from mutable prose/labels. |

## `GITHUB_TOKEN` versus GitHub App/PAT

| Credential | Advantages | Limits/risks | Decision |
|---|---|---|---|
| Built-in `GITHUB_TOKEN` | Ephemeral, repository-scoped, no stored key, workflow-level permissions, best audit locality. | Current docs and metadata-action README conflict about Dependabot `pull_request` write escalation; cannot add a PR to merge queue; repository token policy is hidden. | **Preferred first**, but only after fork proof. |
| Dedicated GitHub App installation token | Short lived, one-repository installation, explicit permission/audit identity; can support alert lookup or queue if granted. | Key management and token-minting action add attack surface; exact endpoint permissions must be proven; must be a Dependabot secret for a Dependabot-triggered workflow. | **Fallback**, starting with metadata/PR permissions and no administration/bypass; add `contents: write` only if API evidence proves required. |
| Fine-grained PAT | Can be repository/resource scoped. | Tied to a user, longer-lived, lifecycle/audit/offboarding burden. | Avoid unless an App is impossible and security approves a time-boxed PoC. |
| Classic PAT/admin/bypass token | Broad and direct merge power. | High-impact credential compromise and policy bypass. | **Reject.** |

Neither token model is allowed to call an immediate/direct merge endpoint, push `master`, edit rules, dismiss reviews, approve the PR, or satisfy required statuses.

## Merge-queue implications

Do not enable a queue in the initial design:

* There is no `merge_group` trigger in checked-in workflows at the baseline.
* GitHub requires workflows that report required queue checks to handle `merge_group`; otherwise the required check is never reported and the queued merge fails [G3].
* GitHub’s current Dependabot automation page says built-in `GITHUB_TOKEN` cannot add a PR to a merge queue; use a PAT or App token with merge permission if a queue is required [G1]. Prefer the App.
* External Prow checks must also recognize GitHub’s queue branches/SHAs, or an explicit migration away from Tide must occur.

Queue rollout is a separate project: enumerate exact required contexts and source IDs, add equivalent `merge_group: {types: [checks_requested]}` coverage, prove the same context names on synthetic queue SHAs, use squash, set “only merge non-failing PRs,” then repeat all deny tests.

## Fork PoC status and exact limits

The authorized fork `aholten/pipelines` remains at `a84029e6c40f209aa49d7c6d7f2cfa3cf11c3ec4`, **212 commits behind and 0 ahead** of upstream `1630f8063b3434217f54da67aea221b910ea238b`. A no-force fast-forward was locally proven, but push failed HTTP 403 for `hermes-selfhost-bot[bot]`. API permissions for that fork were all false. Therefore:

* no synchronized SHA exists in the fork;
* no branch, PR, workflow run, issue, or setting was created or changed;
* no settings need restoration;
* there are no fork run URLs/IDs to cite;
* no fork result validates upstream branch rules, Tide/Prow, token behavior, or hidden settings.

A future PoC is valid only after a credential with `aholten/pipelines` push/admin rights fast-forwards `master` and records the exact synchronized SHA. Fork validation can prove workflow syntax, identity/metadata/path decisions, token behavior, auto-merge pending/cancel behavior, and kill switch. It **cannot** reproduce Kubeflow org rules, hidden legacy protection, production Tide/Prow, external CI credentials/runners, or administrator bypasses.

The operator also identified `aholten/oss-test-infra` as the mechanism used to make Dependabot PRs run CI automatically. That source-first audit is tracked in `t_db33e4da` and the final fork tracking issue is gated on its handoff. This mechanism may remove the need for manual `ok-to-test` or workflow-run approval, but **CI auto-trigger/approval is not merge authorization** and must not grant, imply, or share merge power.

## Rollout phases

### Phase 0 — policy/export (no repository write)

* Admin exports every matching org/repo ruleset and legacy branch-protection rule, required checks and App IDs, strictness, review settings, bypass actors, Actions token/allowed-action policy, auto-merge setting, queue setting, and Tide config/ownership.
* Tide/Prow owners decide: Tide is sole merger, or GitHub native auto-merge is accepted. Do not run two merger authorities.
* Decide whether security patches are eligible. If excluded, approve the App-backed alert classifier.

### Phase 1 — synchronized fork, report-only

* Fast-forward `aholten/pipelines:master` to an explicitly recorded upstream SHA.
* Add the read-only workflow under a PoC branch.
* Exercise synthetic allow/deny PRs; collect run URLs and JSON summaries.
* Confirm no write token or secret is exposed and no PR code executes.

### Phase 2 — upstream label/report-only

* Land Change A with no write permissions.
* Observe at least two Dependabot cycles or 20 classified PR events, whichever is longer.
* Compare decisions with maintainer judgments and the oss-test-infra audit.
* Optional label-only mode may use a separate low-impact label, but labels remain UX telemetry, never a merge gate.

### Phase 3 — one-candidate native auto-merge canary

* Enable repository auto-merge only after recording original settings and admin/Tide approval.
* Apply Change B for one allow-listed Go package/directory or one PR at a time.
* Prove auto-merge stays pending with failed DCO/CI, clears or is reviewed after synchronize/manual change, and squash-merges only after every effective gate.

### Phase 4 — narrow steady state

* Allow all ungrouped direct-production Go patch candidates within the five manifest directories.
* Review weekly metrics and every denial/failed run.
* Expand to groups/minor/Docker/security only by a new written risk decision and validation matrix run. No automatic expansion.

### Phase 5 — optional merge queue

* Separate project after full `merge_group` CI/Prow proof and App-token review.

## Observability, kill switch, and rollback

Every run must record, without secrets:

* repository, PR number/URL, event actor, author identity, base, head SHA, event action;
* metadata action SHA, ecosystem, dependency type, update type, group, maintainer-changes flag;
* sorted changed-file list and deny reasons;
* policy revision and mode (`report`, `label`, `live`);
* credential identity (`GITHUB_TOKEN` or App slug, never token material);
* native auto-merge request result and link to workflow run.

Operational metrics: candidates, eligible/denied by reason, auto-merge requests, pending duration, failed checks, canceled requests, manual overrides, merges, rollbacks, and false-positive/false-negative review findings.

Kill switch order:

1. Disable `.github/workflows/dependabot-auto-merge.yml` in Actions or set its live mode off.
2. Cancel every pending auto-merge request created by the bot (`gh pr merge --disable-auto <URL>`) after listing and reviewing them.
3. Set repository `allow_auto_merge: false` if incident scope warrants it.
4. Revoke/suspend the GitHub App installation or rotate its private key if used.
5. Revert the workflow PR. Do not weaken CI/rules to unblock pending PRs.

Rollback restores the recorded repository/ruleset settings, removes workflow/telemetry labels if desired, and leaves Dependabot PRs open for normal human/Tide handling. Preserve run logs and audit events for incident review.

## Validation matrix

| Test | Expected result |
|---|---|
| Verified Dependabot, Go direct-production patch, ungrouped, allowed manifest files, unchanged commits | Eligible in report mode; later pending native auto-merge. |
| Human/spoofed author, Dependabot-looking title/branch | Job denied/skipped; no request. |
| Dependabot author but event actor is maintainer after a pushed commit | Deny; no new request; existing request must be reviewed/canceled. |
| Metadata action cannot verify Dependabot-only commits | Fail closed. |
| Draft/converted-to-draft | Deny; existing request reviewed/canceled. |
| Wrong base/repository | Deny. |
| Go minor/major, indirect/development, grouped | Deny. |
| Docker, npm, pip, Actions | Deny. |
| Extra source, script, generated, deployment, release, or `.github/**` file | Deny and name the path. |
| Empty/truncated/paginated file result | Fail closed; pagination test must return all files. |
| DCO fails | Native auto-merge remains blocked; no bypass. |
| Any required CI/Prow check fails/cancels/times out | Remains blocked; no bypass. |
| Duplicate/spoofed required context from wrong App | Effective source-bound rule rejects it. |
| `ci-passed` label forged/stale | Has no effect on workflow authorization or branch rules. |
| Head changes during classification | Old run canceled; new head reclassified. |
| Base advances/PR becomes behind | Strict protection blocks or queue later validates integrated SHA. |
| Auto-merge setting disabled | Live step fails visibly; no fallback direct merge. |
| Built-in token read-only | Fork run fails visibly; select App design, never `pull_request_target` + checkout. |
| Security PR when policy says manual | Alert classifier denies; if classifier is unavailable, live rollout remains disabled. |
| Queue required but no `merge_group` results | Rollout blocked; do not enqueue. |
| Tide rejects/holds PR | No GitHub bypass; investigate owner policy. |
| Kill switch activated with pending requests | Workflow disabled, requests enumerated/canceled, App revoked if used. |

## Unknown administrator/maintainer decisions

1. What is the complete effective `master` policy after all org rulesets and legacy protection are combined?
2. Which exact status contexts and integration IDs are required, and is the intended CI aggregate source-bound and strict?
3. Are reviews, stale-review dismissal, last-push approval, conversation resolution, signed commits, deployments, or code scanning required by hidden policy?
4. Who can bypass rules, and can the proposed bot/App bypass anything? Required answer: the bot must not bypass.
5. Are repository auto-merge and squash merge enabled? Public metadata is null for upstream under this integration.
6. What are default and maximum Actions token permissions, allowed actions, secret policy, runner policy, and fork/Dependabot approval settings?
7. Does current platform behavior honor explicit write permissions for Dependabot `pull_request` runs in this repository?
8. Is Tide the sole merger, can it recognize GitHub native auto-merge, or will owners migrate this narrow class to GitHub?
9. What exact `aholten/oss-test-infra` Prow/trusted-user mechanism auto-runs Dependabot CI, and is it still production-owned/active? Await `t_db33e4da`.
10. Are security updates in or out? If out, approve a read-only Dependabot-alert classifier credential.
11. What minimum human review remains required for unattended dependency changes, particularly control-plane paths?
12. Is merge queue desired? If yes, who owns complete `merge_group` and external Prow coverage?
13. Who owns the kill switch, incident response, App key rotation, and weekly metrics review?

## PR slicing and order

1. **PR 1 — policy and CI-source hardening:** document eligibility/deny policy; protect workflows; SHA-pin actions in the merge-control path; no auto-merge.
2. **PR 2 — report-only classifier:** add Change A and fixture tests; no write permission.
3. **PR 3 — effective-policy correction (admin/config):** strict source-bound required contexts, review semantics, no bot bypass; preserve DCO/squash/Tide requirements. Do not bundle with workflow code.
4. **PR 4 — optional Dependabot grouping adjustment:** create individual Go patch candidates only if maintainers accept volume.
5. **PR 5 — native auto-merge canary:** Change B after fork evidence, admin export, security-policy decision, and Tide sign-off.
6. **PR 6 — steady-state scope expansion:** only after canary evidence; one ecosystem/update class per PR.
7. **Separate project — merge queue:** `merge_group` changes to every required workflow/provider, App credential if needed, then queue policy.

Each PR has an independent rollback and no PR should weaken a required gate to make the next PR pass.

## Decision checklist

Production remains **NO-GO** until all are checked:

- [ ] Effective upstream rules/protection/Actions settings exported and reviewed.
- [ ] Exact strict required contexts and source App IDs recorded.
- [ ] Tide/Prow merger ownership decided; oss-test-infra audit incorporated.
- [ ] Security-update policy and reliable classifier decided.
- [ ] Fork synchronized to recorded upstream SHA.
- [ ] Report-only allow/deny matrix passes in the synchronized fork.
- [ ] Actual Dependabot token behavior proven; App fallback reviewed if needed.
- [ ] No checkout/artifact/cache/PR code in the write-capable control plane.
- [ ] Workflow/action SHAs and workflow-path review protection established.
- [ ] Auto-merge pending/failure/synchronize/kill-switch behavior demonstrated.
- [ ] No merge queue unless `merge_group` CI/Prow evidence is complete.

## Authoritative references

* **[G1]** GitHub, “Automating Dependabot with GitHub Actions” — current `pull_request` native auto-merge example, required status-check warning, and queue-token limitation: https://docs.github.com/en/code-security/tutorials/secure-your-dependencies/automate-dependabot-with-actions
* **[G2]** GitHub, “Troubleshooting Dependabot on GitHub Actions” — default read-only behavior, explicit permission escalation, Dependabot-secret boundary, and rerun semantics: https://docs.github.com/en/code-security/reference/supply-chain-security/troubleshoot-dependabot/dependabot-on-actions
* **[G3]** GitHub, “Managing a merge queue” — required `merge_group` event and queue check behavior: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue
* **[G4]** GitHub, “Secure use reference” — privileged-trigger checkout risk, least privilege, script-injection defense, and immutable full-SHA action pinning: https://docs.github.com/en/actions/reference/security/secure-use
* **[G5]** GitHub, “Managing auto-merge for pull requests” — native auto-merge eligibility and cancellation behavior: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-auto-merge-for-pull-requests
* **[G6]** GitHub, “Available rules for rulesets” — required checks, reviews, queue, and related branch rules: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets
* **[G7]** `dependabot/fetch-metadata` pinned action used by the proposal: https://github.com/dependabot/fetch-metadata/tree/d7267f607e9d3fb96fc2fbe83e0af444713e90b7

## Shareable summary for Jeff

Kubeflow Pipelines should not auto-merge every Dependabot PR merely because visible CI is green. The safe path is a no-checkout, Dependabot-only classifier that initially handles only ungrouped direct-production Go patch updates limited to `go.mod`/`go.sum`, then asks GitHub for native squash auto-merge so DCO, strict required CI, reviews, and the chosen Tide/GitHub merge authority remain in control. Production enablement is blocked until admins export the hidden effective branch/Actions settings, Tide owners decide the merger model, the `aholten/oss-test-infra` CI-auto-trigger mechanism is audited, and a synchronized fork proves token, pending-auto-merge, failure, and kill-switch behavior. No upstream or fork mutation was made during this assessment.
