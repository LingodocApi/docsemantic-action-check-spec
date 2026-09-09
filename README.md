# DocSemantic Spec Check — GitHub Action

Fail your CI when your live API responses drift from their OpenAPI/Postman contracts.

This Action is a **thin client** for the hosted DocSemantic API. It sends one authenticated
request to `POST /api/v1/check` and passes or fails the workflow step based on the result.
All drift detection happens server-side in DocSemantic — this Action contains no analysis logic.

## Usage

```yaml
name: API Contract Check
on: [push, pull_request]

jobs:
  spec-check:
    runs-on: ubuntu-latest
    steps:
      - name: DocSemantic Spec Check
        uses: LingodocApi/docsemantic-action-check-spec@v1
        with:
          api-key: ${{ secrets.DOCSEMANTIC_API_KEY }}
```

Add your key (`dsk_live_…`) as a repository secret named `DOCSEMANTIC_API_KEY`
(**Settings → Secrets and variables → Actions**). Never hard-code the key in the workflow file.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `api-key` | yes | — | Your DocSemantic API key (`dsk_live_…`). Pass via a repository secret. |
| `api-url` | no | `https://docsemantic.com` | Base URL of the DocSemantic API. Override only for self-hosted/staging. |
| `fail-on-drift` | no | `true` | When `true`, the step fails if drift is detected. Set `false` to report without failing. |

## Outputs

| Output | Description |
| --- | --- |
| `passed` | `true` when all checked specs are compliant, otherwise `false`. |
| `specs-checked` | Number of specs evaluated. |
| `total-violations` | Total drift violations across all specs. |
| `summary` | Human-readable result summary. |

## Report without failing the build

Useful when first adopting DocSemantic, so drift is visible in the job summary but does not block merges:

```yaml
      - name: DocSemantic Spec Check
        id: docsemantic
        uses: LingodocApi/docsemantic-action-check-spec@v1
        with:
          api-key: ${{ secrets.DOCSEMANTIC_API_KEY }}
          fail-on-drift: "false"

      - name: Comment drift status
        run: echo "Passed=${{ steps.docsemantic.outputs.passed }} — ${{ steps.docsemantic.outputs.summary }}"
```

## How it works

1. The Action calls `POST /api/v1/check` with your API key.
2. DocSemantic compares the contracts registered for your workspace against the live traffic baselines it has learned.
3. The endpoint returns `200` when everything is compliant, or `422` when drift is detected.
4. The Action surfaces the summary in the job log and the workflow summary panel, and fails the step on drift (unless `fail-on-drift: "false"`).

## Response reference

| HTTP status | Meaning | Action behavior |
| --- | --- | --- |
| `200` | All specs compliant | Step passes |
| `422` | Drift detected | Step fails (or warns if `fail-on-drift: "false"`) |
| `401` | Missing/invalid API key | Step fails with an error |
| `429` | Rate limited | Step fails with an error |

## Requirements

Runs on any runner with `bash`, `curl`, and `jq` — all preinstalled on GitHub-hosted runners
(`ubuntu-latest`, `macos-latest`, `windows-latest` with bash shell).
