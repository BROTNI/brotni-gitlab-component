# Security Guidelines

This document describes security considerations and best practices for using the `brotni-gitlab-component`.

---

## Token handling

**Never log the Brotni API token.** The component accepts a `brotni_token_variable` input that names the CI variable holding the token, rather than accepting the token value directly. This prevents the token from being logged in pipeline output or stored in the component configuration.

Checklist:
- Set `BROTNI_TOKEN` (or your chosen variable name) as a **Masked** CI/CD variable in GitLab.
- Set `BROTNI_TOKEN` as **Protected** if you only want it to run on protected branches and tags.
- Never pass the token as a plain `--token` CLI argument — the component passes it via the environment variable `BROTNI_TOKEN` to `brotni-cli`.
- Do not echo or print the value of `BROTNI_TOKEN` in any custom `before_script` or `after_script`.

---

## Network and endpoint security

- Use HTTPS endpoints for `brotni_api_url` at all times.
- Do not hardcode private or internal API endpoints in the component YAML. Use CI/CD variables instead.
- If the Brotni API is hosted inside a private network, configure the runner network accordingly rather than embedding network credentials in the pipeline.

---

## Least-privilege access

The component only requires the following GitLab CI/CD scopes:

| Scope | Reason |
|-------|--------|
| `read_api` | To read merge request metadata when `publish_status` is enabled |
| `write_api` (optional) | Only if the component posts MR status notes |

Avoid granting broader project or group access tokens to `BROTNI_TOKEN` unless your Brotni API requires them.

---

## Runner and image security

- Pin `brotni_cli_image` to a specific digest or version tag (e.g. `ghcr.io/brotni/brotni-cli:1.2.3`) rather than `:latest` in production pipelines. Using `:latest` risks pulling an unvalidated image.
- Use a trusted runner with restricted network access to prevent data exfiltration.
- If using a shared runner, ensure that masked variables are set correctly, as shared runners may log partial output.

---

## Audit and auditability

The component shell scripts are intentionally thin and auditable:
- All logic lives in `templates/*/template.yml`.
- No external scripts are downloaded at runtime.
- The only external call is to `brotni-cli`, which is provided by the container image.

Review the component templates before including them in sensitive pipelines:
```
templates/submit-candidate/template.yml
templates/wait-for-result/template.yml
```

---

## Supply-chain considerations

- Review the `brotni-cli` container image before use.
- In high-assurance environments, build your own `brotni-cli` image from source and reference it via `brotni_cli_image`.
- Verify the image digest after pulling.

---

## Reporting security issues

If you discover a security vulnerability in this component, please report it responsibly via the repository's security advisory process rather than opening a public issue.
