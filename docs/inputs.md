# Component Inputs Reference

This document describes all inputs accepted by the `brotni-gitlab-component` templates.

The component registers candidates in an existing Brotni **campaign** via
`brotni campaign add-candidate`. Create the campaign first (e.g. with
`brotni campaign create`) and pass its ID as `campaign_id`.

---

## `submit-candidate` inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `brotni_api_url` | string | Yes | `$BROTNI_API_URL` | Base URL of the Brotni API endpoint. |
| `brotni_token_variable` | string | Yes | `BROTNI_TOKEN` | Name of the CI variable that holds the Brotni API token. Resolved at runtime; never logged. |
| `campaign_id` | string | **Yes** | `""` | Campaign ID to register this candidate with. |
| `execution_recipe` | string | No | `""` | Path or URL to the execution recipe that describes how this candidate runs. |
| `artifact_uri` | string | No | `""` | OCI artifact URI (e.g. `registry.example.com/myimage:tag`). |
| `artifact_digest` | string | No | `""` | OCI artifact digest in `sha256:...` format. Recommended alongside `artifact_uri`. |
| `candidate_name` | string | No | `mr-<iid>` or `<project>-<short sha>` | Stable candidate name. Idempotent: re-runs upsert rather than duplicate. |
| `source_kind` | string | No | `""` | `git_change`, `container_image`, or `config_bundle`. Inferred from the artifact inputs when empty. |
| `brotni_cli_image` | string | No | `ghcr.io/brotni/brotni-cli:latest` | Docker image that provides `brotni-cli`. Pin for reproducibility. |

### Dotenv artifacts produced by `submit-candidate`

| Variable | Description |
|----------|-------------|
| `BROTNI_CAMPAIGN_ID` | The campaign the candidate was registered with. |
| `BROTNI_CANDIDATE_ID` | The studio-minted candidate ID. |

---

## `wait-for-result` inputs

`wait-for-result` is a **best-effort, non-blocking** report of the campaign
decision. There is no per-candidate job to poll: scoring is comparative and only
exists once the studio has run the candidates and ingested metrics.

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `brotni_api_url` | string | Yes | `$BROTNI_API_URL` | Base URL of the Brotni API endpoint. |
| `brotni_token_variable` | string | Yes | `BROTNI_TOKEN` | Name of the CI variable containing the Brotni API token. |
| `campaign_id` | string | Yes | `$BROTNI_CAMPAIGN_ID` | Campaign to report the decision for (from `submit-candidate`). |
| `brotni_cli_image` | string | No | `ghcr.io/brotni/brotni-cli:latest` | Docker image that provides `brotni-cli`. |

### Dotenv artifacts produced by `wait-for-result`

| Variable | Description |
|----------|-------------|
| `BROTNI_CAMPAIGN_ID` | The campaign the decision was reported for. |

---

## GitLab CI/CD Variables

| Variable | Description | Recommended settings |
|----------|-------------|----------------------|
| `BROTNI_API_URL` | Base URL of the Brotni API | Protected, not masked (not a secret) |
| `BROTNI_TOKEN` | API token for authenticating with the Brotni API | **Masked and Protected** |
| `BROTNI_CAMPAIGN_ID` | Campaign ID to register candidates with | Plain |

> **Security note:** Always mark `BROTNI_TOKEN` as **Masked** in GitLab CI/CD
> variable settings so it never appears in job logs, even on failure.

See [docs/security.md](security.md) for full security guidance.
