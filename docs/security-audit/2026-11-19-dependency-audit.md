# Dependency and security audit: 2026-11-19

Scope: the Helm chart in `charts/unifi-network-application`, the GitHub Actions workflows in `.github/workflows`, and the pre-commit configuration. The repository has no Dockerfile. Its container images are the UniFi application image (`linuxserver/unifi-network-application`, pinned by digest) and the MongoDB sidecar (`mongo:4.4.30`, pinned by digest).

## Summary

This run could not complete the version refresh. Two environment restrictions blocked it:

1. **Upstream release lookups were refused.** The session's GitHub access is scoped to this repository only. Release lookups for the third-party actions and tools (`actions/*`, `azure/setup-helm`, `helm/*`, `pre-commit/*`, and others) were denied, so no newer versions or commit SHAs were verified.
2. **Registry lookups were refused.** Docker Hub returned `403` from the agent proxy, so the latest `linuxserver/unifi-network-application` tag and digest and the MongoDB LTS tag and digest could not be fetched.

Because of this, **no version or digest pins were changed in this PR**. Changing a SHA without verifying it against the upstream tag would be guessing. The findings below are the verified current state, and the update plan lists what the next run must do.

## What was verified in this run

| Check | Result |
|---|---|
| `pre-commit validate-config .pre-commit-config.yaml` | Pass |
| YAML parse: 3 workflows, `Chart.yaml`, `values.yaml`, `.pre-commit-config.yaml` | Pass |
| Actions pinned to full commit SHA with version comment | Pass (8 unique actions, all SHA-pinned) |
| `semver` helper downloaded and verified with SHA-256 | Pass (in `image-updater.yaml`) |
| App image pinned by multi-arch digest | Present: `sha256:f782b39e…d037` (not re-verified against the registry) |
| MongoDB image pinned by digest | Present: `mongo:4.4.30@sha256:4be76f67…62ef` (not re-verified against the registry) |
| `helm lint` / `helm template` | **Not run**: `helm` is not installed in this environment |
| `pre-commit run --all-files` | **Not run**: hook repositories are cloned from GitHub, which is blocked |

## Findings

Severity follows the impact on a homelab or small-cluster deployment.

### High

**H-1. MongoDB 4.4.30 is end-of-life.** MongoDB 4.4 reached end of life in February 2024 and no longer receives security patches (CWE-1104, use of unmaintained third-party components). The sidecar stores all UniFi controller data.
- Location: `charts/unifi-network-application/values.yaml` (`mongodb.imageTag`).
- Action: move to the newest MongoDB line that the linuxserver UniFi build supports. Check the linuxserver/UniFi compatibility matrix first. A wrong major version can prevent the controller from starting, so this needs a staged upgrade and backup, not a blind bump.

**H-2. Database credentials are stored in plaintext in the chart and rendered into the workload.** CWE-256 (plaintext storage of a password), CWE-798 (hard-coded credentials).
- `values.yaml` holds `mongodb.password` and `mongodb.adminpassword`.
- `deployment.yaml` sets them as plain `env.value` on `MONGO_INITDB_ROOT_PASSWORD` and `MONGO_PASS`.
- `configmap.yaml` writes `pwd: "{{ .Values.mongodb.password }}"` into the init script.
- Action: move credentials into a Kubernetes `Secret` (with an `existingSecret` value option) and reference them with `secretKeyRef`.

### Medium

**M-1. Default placeholder credentials are shipped.** The defaults are `password: "replace-me"` and `adminpassword: "replace-me"` (CWE-1392, use of default credentials). A user who installs without overriding them gets a predictable database password.
- Action: fail rendering when the values are still the defaults, using `required` or an explicit check.

**M-2. `dnsPolicy`, `hostNetwork`, and `dnsConfig` are referenced but not defined.** `deployment.yaml` line 114 renders `dnsPolicy: {{ .Values.dnsPolicy }}`, but `values.yaml` has no `dnsPolicy`. The rendered value is empty (null), which can be rejected by the API server or can silently fall back to a policy you did not choose.
- Existing fix: **PR #6** (`fix/pr1-followup-review-fixes`) adds `dnsPolicy: ClusterFirst`, `hostNetwork: false`, and `dnsConfig: {}` defaults and honors the persistence toggles. It is still open and its base (`845a40f`) is behind `main`. It should be rebased and merged rather than duplicated here.

**M-3. Pods have no hardening.** CWE-250 (execution with unnecessary privileges).
- Neither the app nor the MongoDB pod sets a `securityContext`. There is no `allowPrivilegeEscalation: false`, no capability drop, and no seccomp profile.
- `automountServiceAccountToken` is not set to `false`, so the default service account token is mounted.
- The MongoDB container has no resource limits.
- Note: the linuxserver app image starts as root for its init process, so `runAsNonRoot` may not be achievable for the app container without an image change. Apply the hardening that works for both (privilege escalation off, capabilities dropped, seccomp `RuntimeDefault`, service account token not mounted). The MongoDB container can likely run as non-root.

**M-4. Third-party pre-commit hooks are pinned by mutable tag, not commit SHA.** CWE-494 (download of code without an integrity check). The hooks are `conventional-pre-commit v4.5.0`, `pre-commit-hooks v6.0.0`, `gruntwork helmlint v0.1.30`, and `helm-docs v1.14.2`. A tag can be moved. GitHub Actions in the workflows are already pinned by SHA, which is good, so the hooks should match.
- Action: pin each `rev` to the full commit SHA and keep the tag in a comment.

### Low

**L-1. The image-updater workflow trusts an unverified digest.** CWE-345 (insufficient verification of data authenticity). The digest comes from a Docker Hub API response, fetched with `curl -s` (no `--fail`, no timeout). The workflow only checks that it looks like a `sha256:` string.
- Action: use `curl -fsSL --max-time 30`, and verify the digest with `docker buildx imagetools inspect` before writing it to `values.yaml`.

**L-2. Over-broad permissions in the release workflow.** `release.yaml` requests `id-token: write` for "keyless signing", but no signing step exists. Remove the permission unless signing is added.

**L-3. Personal email addresses in the public chart.** `Chart.yaml` lists maintainer email addresses. These are published in the chart and in the chart index. Consider removing the email fields or using a project alias.

**L-4. `image-updater.yaml` grants `contents: write` at job scope.** This is needed for the PR-creation step. Consider splitting the job so the write permission applies only to that step.

### Informational / positive

- All eight distinct GitHub Actions are pinned by full commit SHA with a version comment (see PR #17).
- `persist-credentials: false` is set on checkout where the job does not need to push.
- The semver helper is downloaded and checked against a SHA-256 before use.
- The app image is pinned by multi-arch index digest and applied only when `image.tag` is empty, so a custom tag cannot be combined with a mismatched digest.
- The MongoDB image is pinned by digest.
- Helm is pinned to `v4.3.1` in `release.yaml`.

## Dependency inventory and status

| Component | Current pin | Status | Notes |
|---|---|---|---|
| `linuxserver/unifi-network-application` | `10.6.109` + digest `sha256:f782b39e…` | **Not verified** | Daily `image-updater` job handles this. PR #14 was closed unmerged on 2026-11-14 (superseded by #17). |
| `mongo` | `4.4.30` + digest `sha256:4be76f67…` | **EOL (H-1)** | Needs a compatibility-checked upgrade. |
| `actions/checkout` | `v7.0.1` (SHA `3d3c42e5…`) | **Not verified** | Upstream release check blocked. |
| `actions/setup-python` | `v7.1.0` (SHA `6d4fd22b…`) | **Not verified** | Upstream release check blocked. |
| `gabe565/setup-helm-docs-action` | `v1.0.4` (SHA `d5c35bdc…`) | **Not verified** | Upstream release check blocked. |
| `pre-commit/action` | `v3.0.1` (SHA `2c7b3805…`) | **Not verified** | Upstream release check blocked. |
| `azure/setup-helm` | `v5.1.0` (SHA `2f5e7bc6…`) | **Not verified** | Upstream release check blocked. |
| `helm/chart-releaser-action` | `v1.9.0` (SHA `92c6b97f…`) | **Not verified** | Upstream release check blocked. |
| `docker/login-action` | `v4.8.0` (SHA `8aa5831f…`) | **Not verified** | Upstream release check blocked. |
| `peter-evans/create-pull-request` | `v8.1.3` (SHA `3ffccacd…`) | **Not verified** | Upstream release check blocked. |
| Helm (release workflow) | `v4.3.1` | **Not verified** | Upstream release check blocked. |
| `semver` helper (`fsaintjacques/semver-tool`) | `3.4.0`, SHA-256 checked | OK | Checksum enforced. |
| `conventional-pre-commit` | `v4.5.0` (tag) | **Not verified**, tag-pinned (M-4) | |
| `pre-commit-hooks` | `v6.0.0` (tag) | **Not verified**, tag-pinned (M-4) | |
| `gruntwork-io/pre-commit` (helmlint) | `v0.1.30` (tag) | **Not verified**, tag-pinned (M-4) | |
| `norwoodj/helm-docs` | `v1.14.2` (tag) | **Not verified**, tag-pinned (M-4) | |

## Previous update PRs reviewed

| PR | Title | State | Relevance |
|---|---|---|---|
| #17 | pin actions to SHAs, Helm v4.3.1, appVersion 10.6.109 with digest pin | Merged 2026-11-14 | Latest dependency baseline. This audit builds on it. |
| #16 | bump chart-releaser-action to v1.9.0 and conventional-pre-commit to v4.5.0 | Merged 2026-11-05 | Precedent for the action and hook bumps. |
| #15 | bump chart-releaser-action to v1.8.0, pin helm-docs hook to v1.14.2 | Merged 2026-10-29 | Precedent. |
| #14 | bump(unifi-network-application): update app version | Closed unmerged 2026-11-14 | Superseded by #17. |
| #13 | update actions, pre-commit hooks, pin mongo digest | Merged 2026-10-19 | Precedent for the mongo digest pin. |
| **#6** | fix(chart): honor persistence toggles and pin helm-docs rev | **Open** (since 2026-03-12) | Contains the `dnsPolicy` fix (M-2). Stale against `main`. Recommended to rebase and merge. |

## Update plan for the next run

1. Access: run with GitHub release access for the actions and tools listed above, and with registry access for Docker Hub. Without them the version refresh cannot be completed.
2. Bump each action to its latest release and pin to the full commit SHA with the version comment.
3. Bump pre-commit hook `rev`s to the latest release and pin each to a commit SHA (M-4).
4. Bump Helm in `release.yaml` to the latest v4.x.
5. Refresh the app image digest against the registry for the latest stable tag (or let `image-updater` do it).
6. Plan the MongoDB upgrade off 4.4 (H-1) in a separate PR, with a compatibility note and backup steps.
7. Move credentials to a `Secret` (H-2) and fail on default passwords (M-1).
8. Rebase and merge PR #6 for the `dnsPolicy` fix (M-2).
9. Run the full verification: `pre-commit run --all-files`, `helm lint`, `helm template` with default values and with persistence disabled, and `actionlint` on the workflows.

## Verification status

- Run here: `pre-commit validate-config`, YAML parse of all workflow, chart, and config files, and a grep-based inventory of pins.
- **Not run here:** `pre-commit run --all-files` (hook repos need GitHub access), `helm lint`, `helm template` (no `helm` binary), and `actionlint`. These must be run in a session with GitHub and registry access before the report's recommendations are acted on.
