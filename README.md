# Lantern Security Scan Action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Lantern%20Action-blue.svg?logo=github)](https://github.com/marketplace/actions/lantern-security-scan)
[![GitHub Action](https://img.shields.io/badge/action-lantern--action-blue?logo=github-actions)](https://github.com/DhyaanKanoja11/lantern-action)
[![Lantern](https://img.shields.io/badge/security-Lantern-emerald)](https://github.com/DhyaanKanoja11/Lantern)

An official GitHub Action for [Lantern](https://github.com/DhyaanKanoja11/Lantern) to audit running services and Docker containers in CI, blocking pull requests that introduce unintentional wildcard port bindings (`0.0.0.0` or `::`).

## Quick Start

Add Lantern to your GitHub Actions test or integration pipeline:

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Start Docker Services
        run: docker compose up -d

      - name: Run Integration Tests
        run: npm test # or pytest, go test, etc.

      - name: Audit Port Exposures
        uses: DhyaanKanoja11/lantern-action@v1
        with:
          fail-on-wildcard: 'true'
          ignore-ports: '80,443'
```

## Inputs

| Input | Description | Required | Default |
| :--- | :--- | :---: | :---: |
| `version` | Version of Lantern to install (e.g. `latest`, `v0.2.3`) | No | `latest` |
| `fail-on-wildcard` | Fail the step if any service binds to `0.0.0.0` or `::` | No | `true` |
| `ignore-ports` | Comma-separated list of ports to whitelist (e.g. `80,443`) | No | `""` |
| `sarif-file` | Output path to save OASIS SARIF v2.1.0 security report | No | `""` |
| `patch-file` | Output path to save unified Git remediation diff patch | No | `""` |
| `format` | Output format for CLI stdout (`text`, `json`, `sarif`) | No | `text` |

## Outputs

| Output | Description |
| :--- | :--- |
| `has-patch` | Whether any remediation patch was generated (`"true"` or `"false"`) |
| `patch-file` | File path to the generated unified remediation patch |

---

### Automated Remediation & Fix Pull Requests

Lantern can automatically generate standard unified Git diff patches (`git apply` compatible) for all detected wildcard exposures, allowing CI workflows to open remediation PRs automatically:

```yaml
- name: Audit Port Exposures
  id: lantern
  uses: DhyaanKanoja11/lantern-action@v1
  with:
    fail-on-wildcard: 'false'
    patch-file: 'lantern-fix.patch'

- name: Create Autofix Pull Request
  if: steps.lantern.outputs.has-patch == 'true'
  uses: peter-evans/create-pull-request@v6
  with:
    commit-message: 'fix(security): bind Docker Compose ports to localhost'
    title: 'fix(security): remediate wildcard port bindings'
    body: |
      ## Automated Security Fix by Lantern
      Lantern detected exposed ports bound to wildcard interfaces (`0.0.0.0`) and generated this remediation patch.
    branch: 'lantern/autofix'
```

---

### GitHub Code Scanning & SARIF Alert Ingestion

Upload findings directly into GitHub's **Security → Code scanning alerts** tab:

```yaml
- name: Audit Port Exposures
  uses: DhyaanKanoja11/lantern-action@v1
  with:
    sarif-file: 'results.sarif'
  continue-on-error: true

- name: Upload SARIF to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: results.sarif
```

## How It Works

1. Automatically installs the standalone, zero-dependency `lantern` binary onto the runner.
2. Inspects local Linux kernel socket tables (`/proc/net/tcp`, `ss`) and queries running Docker containers.
3. If an unwhitelisted service is exposed on `0.0.0.0` or `::`:
   - Emits native GitHub Actions error annotations directly on the PR view.
   - Exits with a non-zero code to fail the CI job before accidental exposures reach production.
   - Generates unified diff patches (`lantern fix <port> --diff`) when `patch-file` is configured.

## Documentation

For full documentation and local CLI usage, visit the [Lantern repository](https://github.com/DhyaanKanoja11/Lantern).
