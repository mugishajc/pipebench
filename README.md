# pipebench

Reusable GitHub Actions composite actions for the three jobs most services need: build and sign a container image, run language-aware tests with a coverage floor, and gate a Terraform apply behind a plan comment on pull requests.

The rest of the repo is on purpose small. Each action is a single `action.yml` plus a usage example.

## Actions

| Action | What it does |
|---|---|
| [docker-build-push](docker-build-push/action.yml) | Buildx multi-arch build to any registry, Cosign keyless signing with GitHub OIDC, SLSA provenance, GitHub Actions cache. Emits the image ref and digest as outputs. |
| [terraform-plan-apply](terraform-plan-apply/action.yml) | Runs `terraform fmt`, `validate` and `plan`. On a pull request, posts or updates a comment with the plan summary. On a push to the default branch, applies the saved plan. Uses GitHub OIDC to assume an AWS role, so no static credentials. |
| [test-matrix](test-matrix/action.yml) | Sets up Node, Python, Go or Java, installs dependencies, runs tests with coverage, and enforces a minimum coverage floor. |

## Use

Reference by repo and ref:

```yaml
- uses: mugishajc/pipebench/docker-build-push@v1
  with:
    image: ghcr.io/myorg/my-service
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    sign: "true"

- uses: mugishajc/pipebench/test-matrix@v1
  with:
    language: go
    coverage-floor: "70"

- uses: mugishajc/pipebench/terraform-plan-apply@v1
  with:
    working-directory: infra/envs/prod
    aws-region: eu-west-1
    aws-role-to-assume: arn:aws:iam::111122223333:role/github-actions-deploy
```

See `examples/app-with-pipebench/.github/workflows/ci.yml` for a full consumer workflow.

## Required permissions

In the calling workflow:

```yaml
permissions:
  contents: read
  id-token: write   # needed for keyless Cosign signing and AWS OIDC
  packages: write   # needed for GHCR pushes
```

## Design

- **Composite actions, not reusable workflows.** Composites can be used at any point inside a job, so they compose with other steps. Reusable workflows live at the job level and are harder to combine.
- **No shared base image.** Each action pulls only what it needs. The `docker-build-push` action never runs the Terraform setup.
- **Fail fast on config.** `terraform fmt -check` runs before `plan`, so a formatting problem does not wait for an AWS round trip.
- **OIDC everywhere.** AWS credentials and Cosign signing both use GitHub OIDC. There are no long-lived secrets to rotate.

## CI on this repo

- `actionlint` on every workflow and action under the repo
- a small `shellcheck` sweep that extracts the `run: |` blocks from each composite action's bash steps and lints them
