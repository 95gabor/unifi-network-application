# Dependency and security audit: 2026-11-26

Scope: this fork only (`95gabor/unifi-network-application`). The repo is a Helm chart with GitHub Actions workflows and a pre-commit config. It has no Dockerfiles, so the node/alpine base-image guideline does not apply. The container images in use are `linuxserver/unifi-network-application` (app) and `mongo` (embedded DB, from `values.yaml`).

## 1. Previous update PRs reviewed

| PR | State | Content | Status in this audit |
|---|---|---|---|
| #1–#5, #7–#12 | merged | appVersion bumps by image-updater | no action |
| #6 | open | `fix(chart): honor persistence toggles and pin helm-docs rev` (functional fix, not dependency work) | **left open**, mentioned for rebase (see §6) |
| #13 | merged | action/hook bumps, mongo digest pin | superseded by later PRs |
| #14 | closed (superseded by #17) | appVersion 10.6.106 → 10.6.109 | no action |
| #15, #16 | merged | chart-releaser, helm-docs, conventional-pre-commit bumps | no action |
| #17 | merged | SHA pinning of all actions, Helm v4.3.1, app digest pin | baseline for this audit |
| #18 | open | audit doc from 2026-11-19 (`docs/security-audit/2026-11-19-dependency-audit.md`), no pin changes | **superseded by this PR**, closed with a link (see §6) |

## 2. Changes in this PR

| Item | Before | After | Reason |
|---|---|---|---|
| `linuxserver/unifi-network-application` digest (`charts/.../values.yaml` `image.digest`) | `sha256:f782b39e…d037` | `sha256:31d2bc6f…2e8b` | Upstream re-pushed tags `10.6.109` and `latest` on 2026-11-25. The new digest is the current multi-arch index (amd64 `90536adf…`, arm64 `e2c32c72…`), confirmed against the registry manifest. The old digest still resolves but points to an earlier build of the same version. |
| Helm CLI (`release.yaml`, `azure/setup-helm` `version`) | `v4.3.1` | `v4.3.2` | Latest v4 patch release (`helm/helm` tag `v4.3.2`). Helm 3 is out of security support; v4 is in use. |
| Chart version | `0.5.0` | `0.5.1` | Patch bump for the digest refresh and README (version badge and digest row) updated to match. |

## 3. Checked, already current (no change)

GitHub Actions (all pinned to full commit SHA with a version comment, per #17). Latest tag confirmed via `git ls-remote --tags` against upstream:

| Action | Pinned | Latest release | Commit SHA (peeled tag) |
|---|---|---|---|
| actions/checkout | v7.0.1 | v7.0.1 | `3d3c42e5aac5ba805825da76410c181273ba90b1` |
| actions/setup-python | v7.1.0 | v7.1.0 | `6d4fd22b20465ccc0f7354476588e8df7c9ca0e0` |
| gabe565/setup-helm-docs-action | v1.0.4 | v1.0.4 | `d5c35bdc9133cfbea3b671acadf50a29029e87c2` |
| pre-commit/action | v3.0.1 | v3.0.1 | `2c7b3805fd2a0fd8c1884dcaebf91fc102a13ecd` |
| azure/setup-helm | v5.1.0 | v5.1.0 | `2f5e7bc67f1f9972ab16657bb795f36d8f04de0e` |
| helm/chart-releaser-action | v1.9.0 | v1.9.0 | `92c6b97fa9f664492a2592e0d1d80bc6bf378957` |
| docker/login-action | v4.8.0 | v4.8.0 | `8aa5831fb8d48d8f76594c7ddde61ae58019f550` |
| peter-evans/create-pull-request | v8.1.3 | v8.1.3 | `3ffccacd46053766916f087ce41d627c0b8cf2d1` |

Pre-commit hooks (`.pre-commit-config.yaml`, pinned by tag; latest confirmed):

| Hook | Pinned | Latest |
|---|---|---|
| compilerla/conventional-pre-commit | v4.5.0 | v4.5.0 (commit `33b50eb0…`) |
| pre-commit/pre-commit-hooks | v6.0.0 | v6.0.0 (commit `3e8a8703…`) |
| gruntwork-io/pre-commit | v0.1.30 | v0.1.30 (commit `40f924cd…`) |
| norwoodj/helm-docs | v1.14.2 | v1.14.2 (commit `37d3055f…`) |

Other tools:
- semver-tool (image-updater): tag `3.4.0`, the SHA-256 of the downloaded script matches the pinned value (`1ff4a97e…8991`). Latest tag.
- mongo `4.4.30` digest `sha256:4be76f67…62ef` is unchanged (Docker Hub API tag record matches the pin). Registry manifest check was rate-limited (HTTP 429), so Docker Hub's API is the source for this digest.

## 4. Findings

Carried forward from #18 and re-verified against the current tree. None are fixed in this PR, because each one changes runtime behavior and needs a test against a real UniFi deployment or a maintainer decision.

| ID | Sev | CWE | Finding | Recommended action |
|---|---|---|---|---|
| H-1 | High | CWE-1104 (unmaintained component) | `mongo:4.4.30` is end-of-life (Feb 2024), no further CVE fixes. Upstream supported lines on Docker Hub: 7.0.48, 8.0.38 (LTS), 8.3.17, 9.0.8. | Dedicated migration PR: backup of the PVC first, then stepwise `featureCompatibilityVersion` upgrade (4.4 → 5.0 → 6.0 → 7.0 → 8.0). 5.0+ needs AVX on x86 and ARMv8.2-A on arm64, so Raspberry Pi 4 is not supported. Verify the UniFi compatibility matrix first. **Not changed here**: an unattended data migration risks data loss. |
| H-2 | High | CWE-256 / CWE-312 | DB credentials are stored in plaintext in `values.yaml`, environment variables, and the init ConfigMap. | Move to a Kubernetes `Secret` (or `existingSecret` value). |
| M-1 | Medium | CWE-1188 (insecure default) | Default `replace-me` passwords ship in values. | Fail rendering when defaults are used (`required` in the template). |
| M-2 | Medium | CWE-1188 | `dnsPolicy`/`hostNetwork`/`dnsConfig` are referenced by templates but missing from defaults. Fixed in open PR #6, which is stale against `main`. | Rebase and merge #6. |
| M-3 | Medium | CWE-250 (excess privilege) | No pod `securityContext`, no `automountServiceAccountToken: false`, no MongoDB resource limits. | Add restricted defaults. Test first: the linuxserver image runs s6 init and may need specific capabilities or user settings. |
| M-4 | Medium | CWE-829 | pre-commit hooks are pinned by tag, not SHA. | Pin to commit SHAs (values listed in §3). Not done here to keep the diff focused. |
| L-1 | Low | CWE-494 | Digest in `image-updater` is taken from the Docker Hub API, not verified against the registry manifest. | Optional: add a `docker buildx imagetools inspect` check. |
| L-2 | Low | CWE-250 | `id-token: write` in `release.yaml` is not used by any step (keyless signing is not configured). | Remove, or add a signing step. |
| L-3 | Low | CWE-200 (information exposure) | Personal email addresses in `Chart.yaml` maintainers. | Use a role address or GitHub noreply address. |
| L-4 | Low | CWE-250 | Job-level `contents: write` in `image-updater` and `sync-readme`. | Keep the minimum scope needed for push. |

## 5. Verification

Run in this session:
- YAML parse (`yq`) of all workflows, `.pre-commit-config.yaml`, `Chart.yaml`, `values.yaml`: OK. The five files under `templates/` fail a plain YAML parse as expected, because they contain Go-template syntax.
- Pin inventory: every `uses:` reference in `.github/workflows` is a 40-hex SHA with a version comment.
- Tags and SHAs for every action and hook resolved against upstream with `git ls-remote`.
- Image digests checked against the Docker Hub API and, for linuxserver, the registry manifest (`docker-content-digest` header, 200).
- Stale-reference search: no remaining `0.5.0`, `v4.3.1` or old digest references, apart from the intended `Chart.yaml` appVersion.

**Not run in this session** (install of the binaries was denied by the sandbox; no workaround was attempted):
- `pre-commit run --all-files` (includes `helmlint` and `helm-docs` v1.14.2, which regenerates the README)
- `helm lint` and `helm template` for the chart (default, `--set image.tag=…`, `--set image.digest=""`)
- `actionlint` and `zizmor` on the workflows
- Live run of the `release.yaml` Helm v4.3.2 path (runs on `main` only)

Please run these locally or in CI before merging: `pre-commit run --all-files`, `helm lint charts/unifi-network-application`, `helm template charts/unifi-network-application | grep image:`.

## 6. Related PRs

- **#18** (open, audit doc from 2026-11-19): superseded by this report. Closed with a link here.
- **#6** (open, `fix(chart): honor persistence toggles and pin helm-docs rev`): not a dependency PR, not changed here. The helm-docs pin it mentions already landed in #15. Its `.pre-commit-config.yaml` change will conflict with the `helm-docs` `rev` already on `main`; it needs a rebase. Left open for the owner.
- #14, #13, #15, #16, #17: already merged or closed; no action.

## 7. Open questions for the maintainer

1. Approve the MongoDB 4.4 → 8.0 migration plan (H-1) and pick a maintenance window.
2. Approve restricted pod defaults (M-3) after a test deployment.
3. Confirm whether `Chart.yaml` maintainer emails should be replaced (L-3).
