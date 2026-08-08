# Kubeflow Pipelines uv / `pyproject.toml` migration mission report

**Decision status:** proceed only as a staged, parity-first migration; do not merge or revive the existing broad PR as-is.  
**Evidence cutoff:** 2026-08-08T04:47:22Z.  
**Verified upstream baseline:** `kubeflow/pipelines:master` at [`1630f8063b3434217f54da67aea221b910ea238b`](https://github.com/kubeflow/pipelines/commit/1630f8063b3434217f54da67aea221b910ea238b).  
**Scope:** research and migration design only. No upstream files, issues, PRs, settings, or workflows were modified.

## Executive summary

The migration requested by [issue #12686](https://github.com/kubeflow/pipelines/issues/12686) is still open and is not present on the current default branch. The first implementation, [#12770](https://github.com/kubeflow/pipelines/pull/12770), merged on 2026-03-06 and was reverted about eight hours later by [#12979](https://github.com/kubeflow/pipelines/pull/12979). The successor, [#13085](https://github.com/kubeflow/pipelines/pull/13085), contains useful design and test evidence but is 110 files wide, diverged from current `master`, and currently reports `mergeable=false`, `mergeable_state=dirty`, and `rebaseable=false`.

A source re-check corrects one important point in the parent audit: the #13085 head has **205 GitHub check runs, 204 successful and one skipped**, not merely an untested head. Its legacy combined commit status is still `tide=pending`. Those checks are encouraging historical evidence for commit `359a9abb...`, but they do not certify a conflict-resolved result on the current default branch, which is 26 commits ahead of that PR head while the PR head has four commits not in `master`.

The recommended end state is one non-published root uv workspace, one committed root `uv.lock`, and four PEP 621 member projects: `kfp-pipeline-spec`, `kfp-server-api`, `kfp`, and `kfp-kubernetes`. Use `setuptools.build_meta` for the first conversion to minimize behavioral change; a build-backend switch can be reviewed separately. Preserve each distribution's public dependency metadata and version contract. Use uv workspace sources only for local resolution. Do not absorb unrelated Python services or generated v1 client packaging into this workspace.

The safest delivery is a sequence of independently reversible PRs: first artifact-parity tests and policy, then the additive root workspace, then one package at a time in dependency order, followed by release automation, CI/docs/Docker conversions, and only then legacy-file removal. Generated protobuf and OpenAPI workflows are build prerequisites, not capabilities that uv replaces.

## 1. Verified current-state package and workflow map

### 1.1 Repository-level facts

At `1630f806...`:

- There is no root `pyproject.toml` and no root `uv.lock`.
- The only tracked `pyproject.toml` is the independent [`release/pyproject.toml`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/release/pyproject.toml).
- Six `setup.py` files are tracked. Four are the intended workspace members; the generated v1 client and Google Cloud components package are out of scope.
- Twenty-six `requirements*.in` / `requirements*.txt` files are tracked repository-wide. Only a subset belongs to the four-package migration.
- A current-source scan found 44 `pip install`, `setup.py`, `python -m build`, or `pip-compile` command occurrences across ten SDK-related GitHub workflows/actions. This is a blast-radius indicator, not a claim that every pip use should migrate.

### 1.2 Intended workspace distributions

| Distribution | Current source of truth | Current version / Python | Relationships and packaging constraints |
|---|---|---|---|
| `kfp-pipeline-spec` | [`api/v2alpha1/python/setup.py`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/api/v2alpha1/python/setup.py) + `requirements.in` | `2.15.2`; Python `>=3.9` | Namespace package under `kfp.*`; protobuf `>=6.31.1,<7.0`; generated `pipeline_spec_pb2.py` is created by the proto toolchain and is intentionally cleaned from source. |
| `kfp-server-api` | Generated [`backend/api/v2beta1/python_http_client/setup.py`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/backend/api/v2beta1/python_http_client/setup.py) | `2.17.0`; no explicit `python_requires` in current generated setup | OpenAPI-generated package. Its directory is deleted and recreated by the build script. The generator template, generated metadata, requirements/tox compatibility, and build command must move together. |
| `kfp` | [`sdk/python/setup.py`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/sdk/python/setup.py), `kfp/version.py`, `requirements.in` | `2.15.2`; Python `>=3.9` | Requires `kfp-pipeline-spec>=2.15.0,<3` and `kfp-server-api>=2.15.0,<3`; optional `kubernetes` extra pins `kfp-kubernetes==<kfp version>`; console scripts `kfp` and `dsl-compile`; registry JSON package data must survive. |
| `kfp-kubernetes` | [`kubernetes_platform/python/setup.py`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/kubernetes_platform/python/setup.py), package `__init__.py`, `requirements.in` | `2.15.2`; Python `>=3.9` | Shared `kfp.*` namespace; requires `kfp>=2.14.5,<3`; generated protobuf module; current protobuf lower bound is `6.33.5`, intentionally not identical to pipeline-spec's current `6.31.1`. |

A shared lock must not flatten or silently rewrite these published bounds. In particular, the current version skew (`kfp-server-api` 2.17.0 versus the other three at 2.15.2), exact optional SDK/Kubernetes coupling, Python 3.9 markers for `requests`/`urllib3`, and differing protobuf lower bounds are public compatibility facts.

### 1.3 Generation, build, release, and workflow paths

| Surface | Current behavior | Migration implication |
|---|---|---|
| Pipeline-spec generation | [`api/Makefile`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/api/Makefile) fetches protos, uses a containerized generator, installs `requirements.txt`, runs `generate_proto.py`, then `setup.py sdist` and `pip wheel`. | Keep generation before build. Replace only environment/install/build steps after generated-module and artifact parity pass. |
| Kubernetes package generation | [`kubernetes_platform/Makefile`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/kubernetes_platform/Makefile) uses generated protos, requirements, `PIP_FIND_LINKS`, `setup.py sdist`, and `pip wheel`. | Workspace sources can eventually replace local wheel ordering, but only after local-member resolution and generated artifact tests pass. |
| Server API generation | [`build_kfp_server_api_python_package.sh`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/backend/api/build_kfp_server_api_python_package.sh) `rm -rf`s the output directory, regenerates it from `python_http_client_template`, and runs `python3 setup.py --quiet sdist`. | A hand-maintained generated `pyproject.toml` is invalid. Update the OpenAPI template/source and generated-file validation first. Keep or migrate tox's requirement inputs in the same PR. |
| SDK and release versioning | [`release/kfpr/steps.py`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/release/kfpr/steps.py#L459-L470) edits two Python version files, two `setup.py` constants, and three requirements relationships. | Release automation and its tests are a hard migration gate, not cleanup. Every new version/dependency source must be updated atomically. |
| Publishing | [`publish-packages.yml`](https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/.github/workflows/publish-packages.yml) builds four packages on Python 3.9, runs `twine check`, uploads dry-run artifacts, and retains trusted PyPI publishing. | Change the artifact producer, not `twine check` or trusted publishing. Preserve package-specific output directories and dry-run artifacts. |
| SDK CI | `kfp-sdk-tests.yml`, `kfp-sdk-unit-tests.yml`, `kfp-sdk-client-tests.yml`, `sdk-yapf.yml`; composite actions `kfp-k8s`, `protobuf`, and `test-and-report` contain pip/setup calls. | Translate job by job with minimal dependency groups and complete path triggers for root/member metadata and lock changes. |
| Docs | `.readthedocs.yml`, `readthedocs-builds.yml`, `docs-freshness.yml`, `docs/sdk/build_docs_locally.sh`, `sdk/CONTRIBUTING.md`. | Hosted and local docs need explicit uv installation/sync and clean-clone proof; contributor docs cannot remain pip-only after cutover. |
| Docker | Backend, metadata-writer, visualization, and proxy images have independent Python surfaces. | Do not make every service a workspace member. Apply uv only where the image consumes the workspace; retain the visualization requirements exception until its known conflict is separately resolved. |

### 1.4 Explicitly out of the initial workspace

- `backend/api/v1beta1/python_http_client`: deprecated/generated v1 package and same distribution identity concern; it cannot be a second simultaneous `kfp-server-api` workspace member.
- `components/google-cloud`: independent package with Python `>=3.8` and extensive data files.
- `release/pyproject.toml`: already an independent package; update its orchestration code, not its membership.
- Backend services, visualization, metadata writer, proxy, profile controller, integration-test requirements, and docs-only environments unless a separate owner-approved migration includes them.

## 2. Issue #12686 and related work

| Item | Verified disposition at cutoff | Decision relevance |
|---|---|---|
| [#12686](https://github.com/kubeflow/pipelines/issues/12686) | Open; titled “chore(sdk): Migrate Python dependency management to uv workspaces”; last updated 2026-07-17. | Still the requirements anchor. Do not close it based on reverted or stale branch work. |
| [#12770](https://github.com/kubeflow/pipelines/pull/12770) | Merged as `2d53121b...`; 69 commits, 67 files, +5,262/-1,159. | Historical implementation and test ideas only; restoring it wholesale repeats the risk profile. |
| [#12979](https://github.com/kubeflow/pipelines/pull/12979) | Reverted #12770 as `d2192d24...` on the same day. | Authoritative evidence that merge-time checks did not cover all operational requirements. |
| [Maintainer reopening comment](https://github.com/kubeflow/pipelines/issues/12686#issuecomment-4022732974) | Requires dependency-sensitive workflow triggers, Docker uv conversion, and requirements cleanup, with visualization explicitly excepted. | These are acceptance requirements. Generated-client/tox exceptions need an equally explicit policy rather than silent deletion. |
| [#13085](https://github.com/kubeflow/pipelines/pull/13085) | Open; head `359a9abb...`; 110 files, +7,847/-1,258; `mergeable=false`, dirty, not rebaseable. Relative to current `master`, the head is four commits ahead and 26 behind. | Valuable reference and a source of test evidence, but not a merge vehicle without conflict resolution, scope review, and fresh final-head checks. |

### Reconciled check evidence for #13085

GitHub currently exposes 205 check runs for `359a9abb...`: 204 `success`, one `skipped`, and no failed/cancelled checks. The combined legacy status endpoint separately reports one pending `tide` context. This supersedes the narrower parent statement that only a pending status existed. It does **not** change the decision: those checks certify that old PR head, not a rebase onto `1630f806...`, and GitHub currently says the PR cannot merge cleanly.

## 3. Recommended target uv workspace architecture

```text
/pyproject.toml                                      # non-published workspace + groups
/uv.lock                                             # one committed workspace lock
/api/v2alpha1/python/pyproject.toml                  # kfp-pipeline-spec
/backend/api/v2beta1/python_http_client/pyproject.toml # kfp-server-api (generated)
/sdk/python/pyproject.toml                           # kfp
/kubernetes_platform/python/pyproject.toml           # kfp-kubernetes
```

### Root contract

- Root has no `[project]` and no `[build-system]`; it must not create or publish a synthetic package.
- `[tool.uv.workspace]` ultimately lists exactly the four members.
- Workspace source mappings resolve `kfp`, `kfp-pipeline-spec`, `kfp-server-api`, and `kfp-kubernetes` locally while member `[project.dependencies]` retain normal PyPI-facing version constraints.
- Commit one root `uv.lock`; never commit member lockfiles.
- Put repository-only tooling in PEP 735 groups such as `test`, `lint`, `docs`, and `release`. Prefer minimal job-specific groups over an all-purpose environment.
- Pin the uv version in CI and document both the pin/update process and minimum developer version.
- Model all supported Python resolution environments, at minimum the tested Python 3.9 and 3.13 endpoints. Do not raise package Python floors as part of this migration.

### Member contract

Use PEP 621 metadata with `setuptools.build_meta` initially. Preserve names, versions, classifiers, scripts, extras, namespace discovery, package data, dependency markers/bounds, and sdist/wheel contents. Reuse existing version files for `kfp` and `kfp-kubernetes`; select a release-owned version source for pipeline-spec and generated server API that `kfpr` and the OpenAPI template can update deterministically.

A later backend-change PR may evaluate hatchling or `uv_build` after parity is established. Combining dependency-manager, metadata, and backend behavior changes obscures regressions and weakens rollback.

### Requirements policy

Classify every retained requirements file as one of:

1. **Derived export:** generated with `uv export --frozen`; CI checks zero drift.
2. **Generator/tool compatibility:** owned by an explicit template or tox path, with a documented synchronization command.
3. **Approved standalone exception:** visualization is the currently named exception.
4. **Out of scope:** belongs to a non-workspace service/package.

Only delete a file when its last caller is removed in the same PR. “One lock” means one source for the four-member environment; it does not justify silently deleting unrelated service inputs.

## 4. Phased migration sequence, file scope, and rollback points

| Phase | File-level scope and gate | Rollback point |
|---|---|---|
| 0. Freeze behavior | Add repeatable artifact-parity tooling/tests around the four current builds; capture wheel/sdist filenames, `METADATA`, requirements/extras, entry points, package data, clean-venv imports, and version mutation. Cover `release/kfpr/steps.py` tests. | Test-only revert; no production command changes. |
| 1. Add workspace contract | Add root `pyproject.toml`, pinned uv setup, initial `uv.lock`, lock-check workflow, dependency/requirements policy. Start with only converted members, expanding membership per phase. Keep all legacy commands live. | Revert root files/check; legacy builds remain authoritative. |
| 2. Convert pipeline-spec leaf | Add `api/v2alpha1/python/pyproject.toml`; update `api/Makefile` only after `generate_proto.py` + wheel/sdist parity pass; update release version source/tests. | Keep/reinstate setup/requirements compatibility path; root workspace can remain. |
| 3. Convert generated server API | Update `backend/api/v2beta1/python_http_client_template`, generator config, generated package metadata, `build_kfp_server_api_python_package.sh`, generated-file validation, tox/requirements policy. Regenerate from a clean tree. | Revert template and generated output together; never leave a hand-edited generated pyproject. |
| 4. Convert SDK | Add `sdk/python/pyproject.toml`; preserve dynamic version, scripts, extras, JSON data and bounds; add workspace sources for leaf members; update `sdk/Makefile`, focused SDK jobs, and `release/kfpr/steps.py` plus tests. | Keep legacy requirement/setup path until package tests and release rehearsal pass; revert SDK callers independently. |
| 5. Convert Kubernetes member | Add `kubernetes_platform/python/pyproject.toml`; preserve namespace/version/dev behavior; update Makefile and `.github/actions/kfp-k8s/action.yml`; remove `PIP_FIND_LINKS` only after local workspace resolution proves equivalent. | Revert this member's build/install path without disturbing the first three members. |
| 6. CI/docs/release cutover | Translate SDK workflows/actions, `.readthedocs.yml`, docs scripts, `sdk/CONTRIBUTING.md`, publish dry runs, and applicable Docker consumers. Add complete metadata/lock trigger paths. | Revert one workflow/action/image class at a time; pyprojects and legacy compatibility remain available. |
| 7. Cleanup | Repository-wide caller search; delete in-scope `setup.py`, `requirements*`, update scripts, and `MANIFEST.in` only after parity and release gates are green. Preserve documented exports/exceptions/out-of-scope files. | Each deletion travels with its final caller removal, enabling narrow revert. |

## 5. CI/build/release/test command matrix: before → after

| Purpose | Current command/pattern | Target command/pattern | Required safeguard |
|---|---|---|---|
| Install uv | pip/build tools installed ad hoc | `astral-sh/setup-uv` pinned to an approved version | Renovation policy and checksum/action pin review. |
| Validate declarations | `pip-compile` and committed requirement outputs | `uv lock --check` | Run on root/member `pyproject.toml` and `uv.lock` changes. |
| Intentional dependency update | edit `.in`; run `pip-compile` | edit PEP 621/group declarations; run `uv lock`; review both declaration and lock | Never let ordinary CI auto-update lock. |
| CI environment | `pip install -r ...` / repeated `pip install` | `uv sync --locked --package <member> --group <minimal-group>` | Python 3.9 and 3.13 endpoint lanes; group least privilege. |
| Run tests/lint | direct `pytest`, `yapf`, etc. after pip install | `uv run --locked --package <member> <tool>` after explicit sync | Preserve existing arguments, reports, coverage, and working directory. |
| Editable development | `pip install -e .[dev]` | `uv sync --locked --package <member> --group test` | Clean-clone developer exercise and local-source assertion. |
| Build normal member | `python setup.py sdist`, `pip wheel`, or `python -m build` | `uv build --package <distribution>` | Compare wheel and sdist metadata/content with baseline. |
| Pipeline-spec | `make -C api python` invokes pip/setup | generation remains `make -C api python`; package build becomes `uv build --package kfp-pipeline-spec` | Generated module must exist in wheel and import in clean venv. |
| Kubernetes member | Makefile generation + `PIP_FIND_LINKS` + setup | generation remains; `uv build --package kfp-kubernetes` with workspace sources | Prove imports use local workspace members, not PyPI fallback. |
| Server API | generator script + `python3 setup.py --quiet sdist` | generator emits pyproject; `uv build --package kfp-server-api` | Clean regeneration produces no unexplained diff; tox/export policy passes. |
| Consumer artifact test | `pip install dist/*.whl` | retain an isolated consumer test using `uv pip install` or pip | Workspace editable tests cannot replace published-wheel behavior. |
| Package validation | `twine check dist/*` | unchanged `twine check dist/*` | uv builds; twine still validates/publishes. |
| Publish | `pypa/gh-action-pypi-publish` | unchanged, with uv-produced package directory | Preserve trusted publishing, dry-run artifact uploads, and package selection. |
| Release versioning | `kfpr` edits version files, setup constants, requirements | `kfpr` edits approved version/dependency sources; `uv lock`; builds all four | Unit tests, dry run, clean tagged-checkout rehearsal, exact pin checks. |
| Docs | pip requirements / hosted RTD setup | pinned uv + locked docs group + `uv run` Sphinx | Local and hosted clean builds. |
| Docker workspace consumer | copy requirements; pip install | install pinned uv; copy root/member pyprojects + lock; `uv sync --frozen`/`--locked` as approved | Image build and runtime smoke; do not convert unrelated images implicitly. |

CI should sync with `--locked`; release/container steps may use `--frozen` when the explicit intent is to reject project/lock mutation and avoid project discovery changes. The maintainers should standardize the exact flag policy and test it with the pinned uv version.

## 6. Risk, unknown, and maintainer-decision ledger

| Item | Severity | Known evidence | Closure / decision owner |
|---|---:|---|---|
| Existing successor is non-mergeable and stale | Critical | #13085 is dirty, four ahead/26 behind current master | Rebase or reimplement as sliced PRs; fresh final-head review and checks. |
| A previously merged migration was reverted | Critical | #12770 → #12979 | Treat this as a new migration with explicit regression and rollback gates. |
| Generated OpenAPI metadata can be overwritten | Critical | Server API script deletes/recreates output | Generator-template owner approves source of truth; clean regen/build/test. |
| Proto generation depends on external/container toolchain | High | API/Kubernetes Makefiles and generators | Run both prebuilt and source-accurate paths where required; generated-file checks. |
| Requirements exception policy is ambiguous | High | Issue requests removal except visualization; generated client/tox may still need files | Maintainers approve a per-file derived/generator/exception/out-of-scope ledger. |
| Lock works on all supported Python versions/platforms | High | Current packages promise >=3.9; CI spans 3.9/3.13; local prototype only used 3.13 | Locked sync/build/test on 3.9 and 3.13 and supported runners. |
| Public artifact metadata drift | High | Version skew, exact optional pin, markers, shared namespace, package data | Machine-readable old/new artifact diff and clean consumer installs. |
| Release version mutation and publish ordering | High | `kfpr` edits multiple legacy locations; publish jobs are separate | Maintainer decision on ordering/dependencies; full dry-run and tagged rehearsal. |
| CI trigger completeness | High | Explicit reopening concern | Path-filter inventory proving every Python consumer runs on all relevant metadata/lock changes. |
| Docker conversion boundary | High | Explicit reopening concern; visualization exception | Image-owner inventory and per-image build/smoke; define non-workspace services. |
| Build backend choice | Medium | #13085 uses hatchling; current code is setuptools | Recommendation: retain setuptools initially; maintainers approve any later backend switch separately. |
| uv version and lock-update governance | Medium | Prototype passed on uv 0.11.6 | Maintainers choose pin cadence, action pinning, Dependabot ecosystem, and review policy. |
| Incremental versus atomic delivery | Medium | Original issue described broad atomic migration; broad attempts are 67/110 files | Maintainers approve sliced delivery with compatibility overlap and a final cleanup gate. |
| Fork CI fidelity | Medium | `aholten/pipelines` lacks upstream org/Tide equivalence | Treat fork results as technical PoC evidence, never upstream policy approval. |

## 7. Bounded fork proof-of-concept plan and acceptance criteria

The operator-owned fork is [`aholten/pipelines`](https://github.com/aholten/pipelines). At cutoff its `master` is `a84029e6...`, **212 commits behind and zero ahead** of upstream `1630f806...`. Any PoC must first fast-forward the fork to the exact upstream baseline and record that SHA. This report did not push, open a PR, or run a fork workflow.

### PoC boundary

1. Fast-forward fork `master` to verified upstream `master`; no force push and no mutation of `kubeflow/pipelines`.
2. Create a fork-only feature branch pinned to the synchronized SHA.
3. Implement Phase 0 parity tooling plus the additive root workspace and **only `kfp-pipeline-spec`** as the first member. Keep existing pip/setup paths callable; do not delete requirements or modify publish credentials.
4. Pin uv, commit the root lock, add fork-local lock/parity checks, and translate a narrowly scoped pipeline-spec build lane.
5. If the leaf PoC passes, extend on separate branches/PRs in the slicing order below; do not infer that fork success satisfies upstream Tide, protected-branch, secret, or trusted-publishing policy.

### PoC acceptance criteria

- Fork baseline equals the recorded upstream SHA before changes.
- `uv lock --check` succeeds and `uv sync --locked` succeeds on Python 3.9 and 3.13.
- Proto generation completes, `uv build --package kfp-pipeline-spec` produces wheel and sdist, and a clean environment imports the generated module from the wheel.
- Old/new artifacts have equivalent name, version, `Requires-Python`, `Requires-Dist`, namespace contents, licenses, and required data; any allowed difference is documented.
- Existing focused pipeline-spec tests pass through both legacy and new paths during overlap.
- Generator/source-accurate run leaves no unexpected tracked diff.
- Lock changes trigger the PoC jobs; an intentionally stale lock fails the lock check.
- Reverting root workspace + member conversion restores the unchanged legacy path without manual repair.
- No upstream mutation occurs, and the fork PR text states that fork evidence is not upstream approval.

Docker-backed generation could not be executed in the parent research environment because its Docker daemon was unavailable. It is therefore a mandatory PoC gate, not presumed evidence.

## 8. Recommended PR slicing and order

1. **Policy and parity harness:** artifact snapshots/comparison, requirements classification, uv pin proposal, no build cutover.
2. **Additive root workspace:** non-published root, lock, lock check, contributor opt-in; no legacy deletion.
3. **`kfp-pipeline-spec`:** metadata + proto build parity + release source update.
4. **Generated `kfp-server-api`:** template-first metadata, regeneration, tox/export policy, build parity.
5. **`kfp`:** dynamic version, scripts/extras/data, workspace leaf sources, SDK tests, `kfpr` migration.
6. **`kfp-kubernetes`:** namespace/proto member, local `kfp`, replace `PIP_FIND_LINKS` after proof.
7. **CI and docs lanes:** one coherent job class at a time; complete path triggers and clean-clone docs/developer setup.
8. **Release/publish rehearsal:** all four artifacts, metadata/pip consumer tests, exact dependency coupling, dry-run uploads; preserve trusted publisher.
9. **Applicable Docker consumers:** image-by-image, with visualization explicitly retained as an exception and unrelated services scoped separately.
10. **Cleanup:** delete each legacy file with its final caller; final repository search, generated-file validation, and full integration certification.

Do not stack all ten as one unreviewable chain. PRs 3–6 may depend on the additive root PR, but each package conversion should include its own parity evidence and rollback. Keep the old path until the corresponding package, release, and consumer gates pass. A final upstream implementation should reference #12686 but should not reuse #13085 wholesale without reviewing every change against current master.

## Final go/no-go recommendation

**GO for a fork-local, one-leaf PoC and maintainer design review. NO-GO for merging #13085 as currently reported or closing #12686.**

Production-readiness requires: a conflict-free current-base implementation; approved workspace/requirements/build-backend policies; locked Python 3.9 and 3.13 validation; all four artifact and pip-consumer parity checks; generated OpenAPI/protobuf clean-diff evidence; release/publish dry run; affected Docker and docs builds; complete path triggers; and fresh required checks on the final commit.

## Source registry

### Live GitHub state

- Upstream repository/default branch: https://github.com/kubeflow/pipelines
- Baseline commit: https://github.com/kubeflow/pipelines/commit/1630f8063b3434217f54da67aea221b910ea238b
- Issue: https://github.com/kubeflow/pipelines/issues/12686
- Reopening acceptance-gap comment: https://github.com/kubeflow/pipelines/issues/12686#issuecomment-4022732974
- Original migration: https://github.com/kubeflow/pipelines/pull/12770
- Original merge commit: https://github.com/kubeflow/pipelines/commit/2d53121b41e1821e270afdc50b8ee262d556cefa
- Revert: https://github.com/kubeflow/pipelines/pull/12979
- Revert commit: https://github.com/kubeflow/pipelines/commit/d2192d24fcde9e12843d05e5fe373eb8d7f5ad43
- Successor: https://github.com/kubeflow/pipelines/pull/13085
- Successor head: https://github.com/kubeflow/pipelines/commit/359a9abbdd0b0afde32899b118b7bac594f1a5aa
- Authorized fork: https://github.com/aholten/pipelines

### Pinned current-source paths

- `sdk/python/setup.py`: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/sdk/python/setup.py
- `sdk/python/requirements.in`: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/sdk/python/requirements.in
- `api/v2alpha1/python/setup.py`: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/api/v2alpha1/python/setup.py
- `api/Makefile`: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/api/Makefile
- `kubernetes_platform/python/setup.py`: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/kubernetes_platform/python/setup.py
- `kubernetes_platform/Makefile`: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/kubernetes_platform/Makefile
- Generated v2 client setup: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/backend/api/v2beta1/python_http_client/setup.py
- Server API generator/build: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/backend/api/build_kfp_server_api_python_package.sh
- Release mutation logic: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/release/kfpr/steps.py
- Publish workflow: https://github.com/kubeflow/pipelines/blob/1630f8063b3434217f54da67aea221b910ea238b/.github/workflows/publish-packages.yml

### Parent evidence synthesized

- Package/workflow inventory: task `t_a8055996` handoff.
- Issue/current-state audit: `/opt/data/kanban/attachments/t_3cbd24d0/uv-migration-audit.md`.
- Staged architecture and local uv mechanics prototype: `/opt/data/kanban/attachments/t_2f02c901/uv-pyproject-migration-blueprint.md`.

The parent two-member throwaway prototype used uv 0.11.6 and successfully ran `uv lock`, `uv lock --check`, `uv sync --all-packages --group test`, built both members, and imported the dependent member. That proves the generic non-project-root/workspace-source mechanics, not KFP's resolver, generators, release flow, or CI.
