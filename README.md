# RSO-ekipa10/.github

Shared files of the RSO-ekipa10 organization (project Chance).

| Path | What it is |
|---|---|
| `profile/README.md` | Organization profile shown on <https://github.com/RSO-ekipa10> (in Slovenian) |
| `.github/workflows/service-ci-cd.yml` | Shared CI/CD pipeline that every service repository calls |
| `.github/workflows/lint.yml` | Checks the workflows of this repository on pull requests and on `main` |
| `workflow-templates/` | Starter workflow for a new service, shown under *Actions > New workflow* in the organization's repositories |
| `PULL_REQUEST_TEMPLATE.md` | Default pull request checklist for repositories without their own |
| `.github/dependabot.yml`, `.github/CODEOWNERS` | Weekly action updates; reviews by the other team member for pipeline changes |

## The shared pipeline

Each service has a short `.github/workflows/ci-cd.yml` with one job, `pipeline`, that calls `service-ci-cd.yml`. The branch ruleset requires the check `pipeline / test`.

| Event | Jobs | What happens |
|---|---|---|
| Pull request | `test` | workflow lint, `npm ci`, lint, type check, unit tests with coverage (reports kept as the artifact `test-reports`), `helm lint` and a schema check of the chart with the dev and prod values, image build without push |
| Push to `main` | `test` → `build` → `deploy` | the image is built once and pushed to Docker Hub as `sha-<full commit SHA>`; Helm deploys exactly that image (by digest) to the `prod` namespace on AKS with `--rollback-on-failure --wait`; `helm test` runs the smoke test; if it fails, the release is rolled back to the previous revision |
| Manual run (`workflow_dispatch`) | `deploy` | redeploys the image of an earlier commit of `main` (input `sha`), without building again; the commit must be on `main` and its image on Docker Hub |

The deploy logs in to Azure with OIDC (no stored cloud password), runs in the GitHub environment `production` (only `main` may deploy), and never runs two deploys of a service at once. Before it changes anything, it stops with the exact fix if an earlier deploy was interrupted and left the Helm release stuck (`pending-*`).

Waiting runs: a newer push to a pull request cancels its older run. On `main` (pushes and manual runs) a newer run never cancels a running one, but GitHub keeps only one waiting run: a newer run replaces the waiting one, so the newest commit wins. A replaced merge is not built (it shows as cancelled); its changes ship with the newer commit. A manual run from another branch has its own group and deploys nothing.

### Application settings

An organization secret named after the service (for example `MARKETS_ENV`, lines `KEY=value`) becomes the Kubernetes Secret `<service>-env`; without it the Secret stays as it is.

- Put only secret settings there; non-secret settings belong in `values-prod.yaml`. The deploy masks every single value in the log, so an ordinary short value would be hidden everywhere in the deploy log.
- The deploy writes a hash of the whole secret (never the values) into the Pod annotation `checksum/app-env`, so the Pods restart exactly when the settings change.
- A rollback (automatic, after a failed smoke test, or a manual run with an older `sha`) restores only the image and the chart. The Secret keeps its current settings: to undo a settings change, change the organization secret back and run the workflow manually.

### What a service repository needs

- `package.json` with the scripts `lint`, `typecheck` and `test` (the test runner runs once and accepts `--coverage`, for example `vitest run`), and a committed `package-lock.json`
- a `Dockerfile` in the repository root
- a public Docker Hub repository `<DOCKERHUB_USERNAME>/<service>`: the cluster and the manual-run image lookup pull it without credentials (set Docker Hub *Settings > Default privacy* to Public before the first push, or create the repository as public by hand)
- a Helm chart in `helm/` with `values-dev.yaml` and `values-prod.yaml`, and in it:
  - the image values `image.repository`, `image.tag` and `image.digest` (the Deployment uses `repository@digest` when the digest is set, otherwise `repository:tag`)
  - `podAnnotations: {}` in `values.yaml`, rendered into the Pod template of the Deployment (`spec.template.metadata.annotations`, with `with .Values.podAnnotations` and `toYaml . | nindent 8`)
  - `envFrom` with the `secretRef` `<service>-env` and `optional: true` (the Secret exists only when the organization secret is set)
  - a `helm test` hook in `templates/tests/` that calls the service's health endpoint; it must not mount `<service>-env` or print its environment or configuration, because `helm test --logs` writes its output to the public deploy log
- organization settings visible to the repository: the variables `DOCKERHUB_USERNAME` and `AZURE_CLIENT_ID`; the secrets `DOCKERHUB_TOKEN`, `AZURE_TENANT_ID` and `AZURE_SUBSCRIPTION_ID` (secrets, so that the public deploy logs mask them) and optionally `<SERVICE>_ENV`; and an environment `production` limited to `main`

## Changing the pipeline

Changes go through a pull request whose check `actionlint` passes and which the other team member reviews (`.github/CODEOWNERS`). Today this is a team convention: this repository has no branch ruleset, so nothing enforces it. Recommended later: a ruleset on the default branch that requires a pull request, the check `actionlint` and a code-owner review, and a tag ruleset that stops `v*` tags from being moved or deleted.

A reviewed change on `main` gets a release tag (`vX.Y.Z`). The service repositories will pin the shared workflow to the commit SHA of a release tag, so that a change reaches a service only when its Dependabot pull request (or a manual bump) is merged. Until the first release tag exists, `markets` calls `@main`, so every merge here reaches it at its next run. The pin, once the tag exists:

1. `gh api repos/RSO-ekipa10/.github/commits/vX.Y.Z --jq .sha` (gives the commit SHA, also for an annotated tag)
2. in the service's `ci-cd.yml`: `uses: RSO-ekipa10/.github/.github/workflows/service-ci-cd.yml@<40-character SHA> # vX.Y.Z`

### Tool versions

Dependabot moves the pinned actions (and, in the service repositories, the pinned shared workflow). It does not see the tool versions in the `env` blocks: `HELM_VERSION`, `KUBECTL_VERSION`, `ACTIONLINT_IMAGE` and `KUBECONFORM_IMAGE` in `service-ci-cd.yml`, and `ACTIONLINT_IMAGE` in `lint.yml`. Bump them by hand, monthly (kubectl also when the cluster is upgraded; Helm stays on 4.x, ADR-047). For an image, look up the digest of the new tag, write `<tag>@<digest>`, and let the `actionlint` check of the pull request pass:

```sh
docker buildx imagetools inspect docker.io/rhysd/actionlint:<tag> --format '{{.Manifest.Digest}}'
docker buildx imagetools inspect ghcr.io/yannh/kubeconform:<tag> --format '{{.Manifest.Digest}}'
```

### Organization Actions policy (optional, not enabled)

SHA pinning and the short list of action owners are a convention checked in review, not an enforced setting. An organization owner can enforce both by hand in *Organization settings > Actions > General > Policies*: choose "Allow RSO-ekipa10, and select non-RSO-ekipa10, actions and reusable workflows", allow GitHub's own actions (`actions/*`, `github/*`), allow the patterns `docker/*` and `azure/*` (plus `hashicorp/*` and `terraform-linters/*` for the infrastructure repository), and tick "Require actions to be pinned to a full-length commit SHA". Turn it on only after every service calls the shared workflow at a commit SHA instead of `@main`, and check that the next runs still start.
