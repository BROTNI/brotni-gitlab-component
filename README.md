# brotni-gitlab-component

A reusable [GitLab CI/CD Component](https://docs.gitlab.com/ee/ci/components/) for integrating GitLab merge requests, commits, and OCI artifact candidates with Brotni-compatible simulation workflows.

**This component does not run simulations itself. It registers GitLab candidates in an existing Brotni simulation campaign and reports the campaign decision in CI.**

---

## What this component does

- Collects GitLab CI metadata (project, commit, merge request) and registers it as a candidate in an existing Brotni **campaign** via `brotni campaign add-candidate`.
- Accepts an execution recipe and OCI artifact references; registration is **idempotent** by candidate name (`mr-<iid>`), so re-runs on later pushes update the candidate instead of duplicating it.
- Exposes `BROTNI_CAMPAIGN_ID` and `BROTNI_CANDIDATE_ID` as dotenv artifacts for downstream jobs.
- Provides a best-effort, non-blocking `wait-for-result` job that reports the current campaign decision.

The component is designed to be thin and auditable. All integration logic lives in `brotni-cli`; the component's YAML is only orchestration glue.

> Create the campaign first (e.g. `brotni campaign create --manifest .brotni/simulation.yaml`) and pass its ID as `campaign_id`. There is no non-campaign candidate in the model, so `campaign_id` is required.

---

## Requirements

| Requirement | Details |
|-------------|---------|
| GitLab version | GitLab 16.0+ (CI/CD Components support) |
| CI variable `BROTNI_API_URL` | Base URL of the Brotni API endpoint |
| CI variable `BROTNI_TOKEN` | API token — **set as Masked and Protected** |
| Runner image | The component pulls `ghcr.io/brotni/brotni-cli:latest` by default |

---

## Usage

### Include the component

Add one or both component includes to your `.gitlab-ci.yml`:

```yaml
include:
  - component: $CI_SERVER_FQDN/brotni/brotni-gitlab-component/submit-candidate@~latest
    inputs:
      brotni_api_url: "$BROTNI_API_URL"
      brotni_token_variable: "BROTNI_TOKEN"
      campaign_id: "$BROTNI_CAMPAIGN_ID"
      execution_recipe: "simulation/recipe.yaml"
```

---

### Register a merge request candidate

Use `submit-candidate` to register the current MR commit as a candidate in the campaign:

```yaml
include:
  - component: $CI_SERVER_FQDN/brotni/brotni-gitlab-component/submit-candidate@~latest
    inputs:
      brotni_api_url: "$BROTNI_API_URL"
      brotni_token_variable: "BROTNI_TOKEN"
      campaign_id: "$BROTNI_CAMPAIGN_ID"
      execution_recipe: "simulation/recipe.yaml"
```

The `submit-candidate` job runs in the `.pre` stage automatically. On MR pipelines the candidate is named `mr-<iid>` (idempotent re-registration). It exposes `BROTNI_CAMPAIGN_ID` and `BROTNI_CANDIDATE_ID` as dotenv artifacts.

See [`examples/basic-mr.gitlab-ci.yml`](examples/basic-mr.gitlab-ci.yml) for a complete example.

---

### Register an OCI image artifact

To register a built container image (pinned by digest), provide `artifact_uri` and `artifact_digest`:

```yaml
include:
  - component: $CI_SERVER_FQDN/brotni/brotni-gitlab-component/submit-candidate@~latest
    inputs:
      brotni_api_url: "$BROTNI_API_URL"
      brotni_token_variable: "BROTNI_TOKEN"
      campaign_id: "$BROTNI_CAMPAIGN_ID"
      artifact_uri: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
      artifact_digest: "$BROTNI_ARTIFACT_DIGEST"
      source_kind: "container_image"
```

Capture the image digest in a `build` job using `docker inspect` or `crane digest`, then pass it via `BROTNI_ARTIFACT_DIGEST` in a dotenv artifact.

See [`examples/oci-artifact.gitlab-ci.yml`](examples/oci-artifact.gitlab-ci.yml) for a complete example.

---

### Report the campaign decision

Include `wait-for-result` after `submit-candidate` for a best-effort, non-blocking report of the campaign decision. There is no per-candidate job to poll — scoring is comparative and only exists once the studio has run the candidates and ingested metrics:

```yaml
include:
  - component: $CI_SERVER_FQDN/brotni/brotni-gitlab-component/submit-candidate@~latest
    inputs:
      brotni_api_url: "$BROTNI_API_URL"
      brotni_token_variable: "BROTNI_TOKEN"
      campaign_id: "$BROTNI_CAMPAIGN_ID"

  - component: $CI_SERVER_FQDN/brotni/brotni-gitlab-component/wait-for-result@~latest
    inputs:
      brotni_api_url: "$BROTNI_API_URL"
      brotni_token_variable: "BROTNI_TOKEN"
      campaign_id: "$BROTNI_CAMPAIGN_ID"
```

`wait-for-result` prints the current campaign decision and exposes `BROTNI_CAMPAIGN_ID` as a dotenv artifact.

See [`examples/wait-for-result.gitlab-ci.yml`](examples/wait-for-result.gitlab-ci.yml) for a complete example.

---

## Required CI/CD variables

Set these at the project or group level in **Settings > CI/CD > Variables**:

| Variable | Required | Description | Recommended settings |
|----------|----------|-------------|----------------------|
| `BROTNI_API_URL` | Yes | Base URL of the Brotni API | Not masked (not a secret) |
| `BROTNI_TOKEN` | Yes | API authentication token | **Masked and Protected** |
| `BROTNI_CAMPAIGN_ID` | Yes | Campaign ID to register candidates with | Plain |

---

## Security best practices

- **Mask `BROTNI_TOKEN`** — set the variable as Masked in GitLab so it never appears in job logs.
- **Protect `BROTNI_TOKEN`** — set it as Protected to restrict use to protected branches and tags.
- **Use HTTPS** — always configure `BROTNI_API_URL` with an HTTPS endpoint.
- **Pin the CLI image** — set `brotni_cli_image` to a specific version tag or digest in production (e.g. `ghcr.io/brotni/brotni-cli:1.2.3`) rather than `:latest`.
- **Audit the templates** — the component scripts in `templates/` are intentionally minimal. Review them before including in sensitive pipelines.
- **Do not log the token** — the component resolves the token by variable name at runtime and never echoes it.

See [`docs/security.md`](docs/security.md) for full security guidance.

---

## How this relates to `brotni-cli`

`brotni-cli` is the command-line tool that handles the actual communication with the Brotni API. This component is a GitLab CI/CD wrapper around `brotni-cli` that:

1. Collects GitLab CI context (commit SHA, MR IID, project path, etc.).
2. Constructs the correct `brotni campaign add-candidate` arguments.
3. Passes the API token securely via environment variable.
4. Exposes the campaign and candidate IDs as GitLab CI dotenv artifacts.

The component does not duplicate `brotni-cli`'s integration logic. Keeping the component thin means that improvements to `brotni-cli` are immediately available without updating the component templates.

---

## Component inputs reference

See [`docs/inputs.md`](docs/inputs.md) for a full reference of all inputs and dotenv artifacts.

---

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
