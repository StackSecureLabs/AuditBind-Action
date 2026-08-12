# AuditBind

AuditBind evaluates pull requests for SOC 2-oriented code and change-management controls, creates tamper-evident JSON evidence, and can send the results to your AuditBind dashboard.

It works directly from GitHub pull-request metadata and file patches. You do not need to check out or build your application before running AuditBind.

## Quick start

### 1. Add your AuditBind API key

In the repository you want to monitor:

1. Open **Settings → Secrets and variables → Actions**.
2. Select **New repository secret**.
3. Name the secret `AUDITBIND_API_KEY`.
4. Paste the API key issued by your AuditBind dashboard.

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
          required_approvals: '1'
          fail_on_critical: 'true'
          comment_on_pr: 'true'

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

## Use one key across multiple repositories

You can store one AuditBind API key as an organization-level GitHub Actions secret:

1. Open the GitHub organization **Settings**.
2. Select **Secrets and variables → Actions**.
3. Create an organization secret named `AUDITBIND_API_KEY`.
4. Grant access to all repositories or only the repositories that should report to AuditBind.

Use the same input in each repository:

```yaml
auditbind_api_key: ${{ secrets.AUDITBIND_API_KEY }}
```

Every run includes the repository owner and name, commit SHA, branch, and pull-request number so the dashboard can separate results by repository.

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

| Input | Default | Description |
| --- | --- | --- |
| `github_token` | GitHub workflow token | Reads pull-request metadata and optionally posts comments. |
| `required_approvals` | `1` | Minimum approving reviews required by CC8.1. |
| `stale_approval_days` | `30` | Approvals older than this are treated as stale. |
| `fail_on_critical` | `true` | Fail the job when a critical finding is detected. |
| `comment_on_pr` | `true` | Create or update an AuditBind pull-request comment. |
| `auditbind_api_key` | Empty | AuditBind dashboard API key. Upload is skipped when omitted. |
| `auditbind_api_url` | `https://api.auditbind.io` | AuditBind API base URL. Usually unchanged. |
| `output_dir` | `./evidence` | Directory for the JSON evidence artifact. |
| `llm_api_key` | Empty | Optional customer-provided LLM key for CC7.1 and CC7.2. |
| `llm_provider` | `auto` | Provider name or automatic detection from the model. |
| `llm_model` | `gpt-4o-mini` | Model used for optional LLM analysis. |
| `llm_base_url` | Empty | Optional custom OpenAI-compatible endpoint. |

## Outputs

| Output | Description |
| --- | --- |
| `verdict` | `PASS`, `FAIL`, `NOT_APPLICABLE`, or `ERROR`. |
| `artifact_path` | Path to the generated evidence JSON file. |
| `artifact_hash` | SHA-256 hash of the evidence artifact. |
| `findings_count` | Findings reported by the primary change-management evaluation. |
| `dashboard_url` | Dashboard URL returned after a successful upload. |

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

## Version pinning

For normal use, pin to the latest compatible major:

```yaml
uses: StackSecureLabs/AuditBind-Action@v1
```

Security-sensitive organizations can pin the action to a full commit SHA.

## Troubleshooting

### No dashboard results

Confirm that the secret is named `AUDITBIND_API_KEY`, the workflow passes it as `auditbind_api_key`, the organization secret can access the repository, and the pull request was not opened from a fork. Local evidence is still generated when dashboard upload is unavailable.

### Pull-request comment is missing

Confirm that `comment_on_pr` is enabled and the workflow grants `pull-requests: write` and `issues: write`.

### The action reports `NOT_APPLICABLE`

AuditBind's change-management evaluation requires pull-request context. Use the `pull_request` trigger shown above.

## Support

Open an issue in this repository for product questions and reproducible bugs. Remove API keys, tokens, proprietary source code, and evidence payloads from logs before posting.

## License

AuditBind Action is available under the [MIT License](LICENSE).
