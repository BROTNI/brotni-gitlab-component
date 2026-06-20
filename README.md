# brotni-gitlab-component

A reusable [GitLab CI/CD Component](https://docs.gitlab.com/ee/ci/components/) for integrating GitLab merge requests, commits, and OCI artifact candidates with Brotni-compatible simulation workflows.

**This component does not run simulations itself. It submits GitLab candidates to a Brotni-compatible simulation workflow and exposes the resulting status in CI.**

---

## What this component does

- Collects GitLab CI metadata (project, commit, merge request) and submits it as a simulation candidate to a Brotni-compatible API via `brotni-cli`.
- Accepts simulation configuration inputs: simulation spec, execution recipe, context spec, and OCI artifact references.
- Optionally waits for simulation completion and exposes `BROTNI_STATUS`, `BROTNI_SCORE`, and `BROTNI_REPORT_URL` as CI/CD variables for downstream jobs.
- Publishes simulation status back to the GitLab merge request when enabled.

The component is designed to be thin and auditable. All integration logic lives in `brotni-cli`; the component's YAML is only orchestration glue.

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
      simulation_spec: "simulation/spec.yaml"
      execution_recipe: "simulation/recipe.yaml"
      candidate_name: "$CI_PROJECT_NAME-$CI_COMMIT_SHORT_SHA"
      campaign_id: "$BROTNI_CAMPAIGN_ID"
```

---

### Submit a merge request candidate

Use `submit-candidate` in your pipeline to automatically submit the current MR commit to the Brotni simulation workflow:

```yaml
include:
  - component: $CI_SERVER_FQDN/brotni/brotni-gitlab-component/submit-candidate@~latest
    inputs:
      brotni_api_url: "$BROTNI_API_URL"
      brotni_token_variable: "BROTNI_TOKEN"
      simulation_spec: "simulation/spec.yaml"
      execution_recipe: "simulation/recipe.yaml"
      candidate_name: "$CI_PROJECT_NAME-mr-$CI_MERGE_REQUEST_IID"
      publish_status: "true"
```

The `submit-candidate` job runs in the `.pre` stage automatically. It exposes `BROTNI_SIMULATION_JOB_ID` as a dotenv artifact for use by `wait-for-result`.

See [`examples/basic-mr.gitlab-ci.yml`](examples/basic-mr.gitlab-ci.yml) for a complete example.

---

### Submit an OCI image artifact

To submit a built container image as the simulation candidate, provide the `artifact_uri` and `artifact_digest` inputs:

```yaml
include:
  - component: $CI_SERVER_FQDN/brotni/brotni-gitlab-component/submit-candidate@~latest
    inputs:
      brotni_api_url: "$BROTNI_API_URL"
      brotni_token_variable: "BROTNI_TOKEN"
      simulation_spec: "simulation/spec.yaml"
      artifact_uri: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
      artifact_digest: "$BROTNI_ARTIFACT_DIGEST"
      candidate_name: "$CI_PROJECT_NAME-$CI_COMMIT_SHORT_SHA"
```

Capture the image digest in a `build` job using `docker inspect` or `crane digest`, then pass it via `BROTNI_ARTIFACT_DIGEST` in a dotenv artifact.

See [`examples/oci-artifact.gitlab-ci.yml`](examples/oci-artifact.gitlab-ci.yml) for a complete example.

---

### Wait for simulation results

Include `wait-for-result` after `submit-candidate` to block the pipeline until the simulation completes:

```yaml
include:
  - component: $CI_SERVER_FQDN/brotni/brotni-gitlab-component/submit-candidate@~latest
    inputs:
      brotni_api_url: "$BROTNI_API_URL"
      brotni_token_variable: "BROTNI_TOKEN"
      simulation_spec: "simulation/spec.yaml"

  - component: $CI_SERVER_FQDN/brotni/brotni-gitlab-component/wait-for-result@~latest
    inputs:
      brotni_api_url: "$BROTNI_API_URL"
      brotni_token_variable: "BROTNI_TOKEN"
      simulation_job_id: "$BROTNI_SIMULATION_JOB_ID"
      poll_interval_seconds: "30"
      timeout_minutes: "90"
```

On completion, the following variables are available via dotenv artifacts:

| Variable | Description |
|----------|-------------|
| `BROTNI_STATUS` | `completed`, `passed`, `failed`, `error`, or `timeout` |
| `BROTNI_SCORE` | Numeric score from the simulation |
| `BROTNI_REPORT_URL` | URL to the full simulation report |
| `BROTNI_CAMPAIGN_URL` | URL to the campaign comparison view, when the candidate belongs to a campaign |

See [`examples/wait-for-result.gitlab-ci.yml`](examples/wait-for-result.gitlab-ci.yml) for a complete example.

---

## Required CI/CD variables

Set these at the project or group level in **Settings > CI/CD > Variables**:

| Variable | Required | Description | Recommended settings |
|----------|----------|-------------|----------------------|
| `BROTNI_API_URL` | Yes | Base URL of the Brotni API | Not masked (not a secret) |
| `BROTNI_TOKEN` | Yes | API authentication token | **Masked and Protected** |
| `BROTNI_CAMPAIGN_ID` | No | Default campaign ID | Plain |

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

1. Collects GitLab CI context (commit SHA, MR IID, project URL, etc.).
2. Constructs the correct `brotni-cli submit` arguments.
3. Passes the API token securely via environment variable.
4. Exposes results as GitLab CI dotenv artifacts.

The component does not duplicate `brotni-cli`'s integration logic. Keeping the component thin means that improvements to `brotni-cli` are immediately available without updating the component templates.

---

## Component inputs reference

See [`docs/inputs.md`](docs/inputs.md) for a full reference of all inputs and dotenv artifacts.

---

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
