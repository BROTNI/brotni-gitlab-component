# Component Inputs Reference

This document describes all inputs accepted by the `brotni-gitlab-component` templates.

---

## `submit-candidate` inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `brotni_api_url` | string | Yes | `$BROTNI_API_URL` | Base URL of the Brotni API endpoint (e.g. `https://api.brotni.example.com`). |
| `brotni_token_variable` | string | Yes | `BROTNI_TOKEN` | Name of the CI variable that holds the Brotni API token. The variable is resolved at runtime; its value is never logged. |
| `simulation_spec` | string | No | `""` | Path or URL to the simulation specification file. |
| `execution_recipe` | string | No | `""` | Path or URL to the execution recipe file. |
| `context_spec` | string | No | `""` | Path or URL to the context specification file. |
| `artifact_uri` | string | No | `""` | OCI artifact URI (e.g. `registry.example.com/myimage:tag`). Use for artifact-based candidates. |
| `artifact_digest` | string | No | `""` | OCI artifact digest in `sha256:...` format. Recommended alongside `artifact_uri`. |
| `candidate_name` | string | No | `$CI_PROJECT_NAME-$CI_COMMIT_SHORT_SHA` | Human-readable name for the simulation candidate. |
| `campaign_id` | string | No | `""` | Brotni campaign ID to associate this candidate with. |
| `wait` | boolean | No | `false` | When `true`, the job waits for simulation completion before exiting. For long-running simulations, prefer using the separate `wait-for-result` component. |
| `publish_status` | boolean | No | `true` | When `true`, publishes simulation status back to the GitLab merge request (if running in an MR pipeline). |
| `brotni_cli_image` | string | No | `ghcr.io/brotni/brotni-cli:latest` | Docker image that provides `brotni-cli`. Pin to a specific version for reproducibility. |

### Dotenv artifacts produced by `submit-candidate`

| Variable | Description |
|----------|-------------|
| `BROTNI_CANDIDATE_SUBMITTED` | Set to `true` when the candidate was successfully submitted. |
| `BROTNI_SIMULATION_JOB_ID` | ID of the created simulation job (set by `brotni-cli`). Used by `wait-for-result`. |

---

## `wait-for-result` inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `brotni_api_url` | string | Yes | `$BROTNI_API_URL` | Base URL of the Brotni API endpoint. |
| `brotni_token_variable` | string | Yes | `BROTNI_TOKEN` | Name of the CI variable containing the Brotni API token. |
| `simulation_job_id` | string | Yes | `$BROTNI_SIMULATION_JOB_ID` | ID of the simulation job to poll. Typically populated from the `submit-candidate` dotenv artifact. |
| `poll_interval_seconds` | integer | No | `30` | Seconds between polling requests to the Brotni API. |
| `timeout_minutes` | integer | No | `60` | Maximum number of minutes to wait before failing the job. |
| `brotni_cli_image` | string | No | `ghcr.io/brotni/brotni-cli:latest` | Docker image that provides `brotni-cli`. |

### Dotenv artifacts produced by `wait-for-result`

| Variable | Description |
|----------|-------------|
| `BROTNI_STATUS` | Final simulation status (`completed`, `passed`, `failed`, `error`, `timeout`). |
| `BROTNI_SCORE` | Numeric score returned by the simulation (present when `BROTNI_STATUS` is `completed` or `passed`). |
| `BROTNI_REPORT_URL` | URL to the full simulation report (present on success). |

---

## GitLab CI/CD Variables

The following CI/CD variables should be defined at the project or group level:

| Variable | Description | Recommended settings |
|----------|-------------|----------------------|
| `BROTNI_API_URL` | Base URL of the Brotni API | Protected, not masked (not a secret) |
| `BROTNI_TOKEN` | API token for authenticating with the Brotni API | **Masked and Protected** |
| `BROTNI_CAMPAIGN_ID` | Default campaign ID (optional) | Plain |

> **Security note:** Always mark `BROTNI_TOKEN` as **Masked** in GitLab CI/CD variable settings to prevent it from appearing in job logs, even on failure.

See [docs/security.md](security.md) for full security guidance.
