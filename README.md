# AuditBind

AuditBind evaluates pull requests for SOC 2-oriented code and change-management controls, creates tamper-evident JSON evidence, and can send the results to your AuditBind dashboard.

It works directly from GitHub pull-request metadata and file patches. You do not need to check out or build your application before running AuditBind.

## Quick start

### 1. Add your AuditBind API key

In the repository you want to monitor:

1. Open **Settings → Secrets and variables → Actions**.
2. Select **New repository secret**.
3. Name the secret `AUDITBIND_API_KEY`.
4. Paste the repository-scoped API key issued by the repository's setup page in AuditBind.

Never place the API key directly in a workflow file.

### 2. Add the workflow

Create `.github/workflows/auditbind.yml`:

```yaml
name: AuditBind

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  audit:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest

    steps:
      - name: Evaluate SOC 2 controls
        id: auditbind
        uses: StackSecureLabs/AuditBind-Action@v1
        with:
          auditbind_api_key: ${{ secrets.AUDITBIND_API_KEY }}
          required_approvals: "1"
          fail_on_critical: "true"
          comment_on_pr: "true"

      - name: Preserve local evidence
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: auditbind-evidence-pr-${{ github.event.pull_request.number }}
          path: evidence/
          if-no-files-found: warn
          retention-days: 30
```

AuditBind evaluates the pull request, posts or updates a summary comment, writes an evidence artifact under `evidence/`, and uploads the result when an AuditBind API key is supplied.

## Keys are scoped to one repository

Issue a separate key for each connected repository. The key is bound to GitHub's stable numeric repository ID, so a key copied to another repository is rejected and cannot write evidence into the wrong project.

You can still manage deployment centrally with reusable workflows or organization automation. Store each issued value only in the matching repository's `AUDITBIND_API_KEY` secret. Organization-wide upload credentials are not part of v1.

Use the regional base URL shown on the repository setup page. US workspaces use `https://ingest.auditbind.com`; EU workspaces use `https://eu.ingest.auditbind.com`. A request sent to the wrong region follows one safe HTTPS redirect.

## What AuditBind evaluates

- **CC6.1** — logical access controls
- **CC6.3** — network security controls
- **CC6.6** — encryption in transit
- **CC6.7** — encryption at rest and secret exposure
- **CC6.8** — secrets management
- **CC7.1** — system monitoring
- **CC7.2** — incident detection and response
- **CC8.1** — pull-request change management
- **CC9.1** — software supply-chain risk

The default checks use GitHub metadata and static analysis. CC7.1 and CC7.2 can optionally use a customer-provided LLM key for additional context-aware analysis.

## Required GitHub permissions

```yaml
permissions:
  contents: read
  pull-requests: write
  issues: write
```

- `contents: read` allows AuditBind to inspect repository and pull-request information.
- `pull-requests: write` and `issues: write` allow AuditBind to create or update its pull-request comment.

Set `comment_on_pr: 'false'` if you do not want pull-request comments.

## Inputs

| Input                 | Default                        | Description                                                  |
| --------------------- | ------------------------------ | ------------------------------------------------------------ |
| `github_token`        | GitHub workflow token          | Reads pull-request metadata and optionally posts comments.   |
| `required_approvals`  | `1`                            | Minimum approving reviews required by CC8.1.                 |
| `stale_approval_days` | `30`                           | Approvals older than this are treated as stale.              |
| `fail_on_critical`    | `true`                         | Fail the job when a critical finding is detected.            |
| `comment_on_pr`       | `true`                         | Create or update an AuditBind pull-request comment.          |
| `auditbind_api_key`   | Empty                          | AuditBind dashboard API key. Upload is skipped when omitted. |
| `auditbind_api_url`   | `https://ingest.auditbind.com` | AuditBind ingest origin or full `/api/v1/evidence` URL.      |
| `output_dir`          | `./evidence`                   | Directory for the JSON evidence artifact.                    |
| `llm_api_key`         | Empty                          | Optional customer-provided LLM key for CC7.1 and CC7.2.      |
| `llm_provider`        | `auto`                         | Provider name or automatic detection from the model.         |
| `llm_model`           | `gpt-4o-mini`                  | Model used for optional LLM analysis.                        |
| `llm_base_url`        | Empty                          | Optional custom OpenAI-compatible endpoint.                  |

## Outputs

| Output           | Description                                                   |
| ---------------- | ------------------------------------------------------------- |
| `verdict`        | `PASS`, `FAIL`, `NOT_APPLICABLE`, or `ERROR`.                 |
| `artifact_path`  | Path to the generated evidence JSON file.                     |
| `artifact_hash`  | SHA-256 hash of the evidence artifact.                        |
| `findings_count` | Total findings reported across every evaluated control.       |
| `dashboard_url`  | Dashboard URL returned after a successful upload.             |
| `upload_status`  | `uploaded`, `duplicate`, `skipped`, or `error`.               |
| `evidence_id`    | Stable cloud evidence record ID.                              |
| `reason`         | Machine-readable reason when no pull-request evaluation runs. |

Example:

```yaml
- name: Print AuditBind result
  if: always()
  run: |
    echo "Verdict: ${{ steps.auditbind.outputs.verdict }}"
    echo "Evidence: ${{ steps.auditbind.outputs.artifact_path }}"
    echo "Dashboard: ${{ steps.auditbind.outputs.dashboard_url }}"
```

## Optional LLM analysis

```yaml
- name: Evaluate SOC 2 controls
  uses: StackSecureLabs/AuditBind-Action@v1
  with:
    auditbind_api_key: ${{ secrets.AUDITBIND_API_KEY }}
    llm_api_key: ${{ secrets.OPENAI_API_KEY }}
    llm_provider: openai
    llm_model: gpt-4o-mini
```

The LLM key is optional. Metadata and static-analysis checks continue to run without it.

## Pull requests from forks

GitHub does not provide repository secrets to workflows triggered by pull requests from forks. In those runs, dashboard and LLM uploads are skipped, while metadata checks and local evidence generation can continue.

Do not switch to `pull_request_target` solely to expose secrets to untrusted fork code.

## Testing against a development environment

Keep a changing tunnel or staging origin in a GitHub Actions repository variable instead of
hardcoding it in the workflow. Create `AUDITBIND_API_URL` under **Settings → Secrets and
variables → Actions → Variables**, then pass it to the Action:

```yaml
with:
  auditbind_api_key: ${{ secrets.AUDITBIND_API_KEY }}
  auditbind_api_url: ${{ vars.AUDITBIND_API_URL }}
```

For a Cloudflare Quick Tunnel, the value is the HTTPS origin only, such as
`https://example.trycloudflare.com`. AuditBind appends `/api/v1/evidence`. The API key remains
a secret and must never be stored as a variable.

## Version pinning

For normal use, pin to the latest compatible major:

```yaml
uses: StackSecureLabs/AuditBind-Action@v1
```

Security-sensitive organizations can pin the action to a full commit SHA.

## Troubleshooting

### No dashboard results

Confirm that the secret is named `AUDITBIND_API_KEY`, the workflow passes it as
`auditbind_api_key`, the key was issued for this exact repository, and the pull request was not
opened from a fork. Local evidence is still generated when dashboard upload is unavailable.

### Pull-request comment is missing

Confirm that `comment_on_pr` is enabled and the workflow grants `pull-requests: write` and `issues: write`.

### The action reports `NOT_APPLICABLE`

AuditBind's change-management evaluation requires pull-request context. Use the `pull_request` trigger shown above.

## Support

Open an issue in this repository for product questions and reproducible bugs. Remove API keys, tokens, proprietary source code, and evidence payloads from logs before posting.

## License

AuditBind Action is available under the [MIT License](LICENSE).
