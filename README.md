# ugs-gha-pr-review

Shared reusable workflow for **automated PR review** across UGS-GIO repos, using Google's
[`run-gemini-cli`](https://github.com/google-github-actions/run-gemini-cli) action (Gemini on
Vertex AI). This repo is **public** so it can be called from public *and* private org repos alike;
it contains no secrets — auth is Workload Identity Federation via org-level Actions variables.

## Add the reviewer to a repo

1. Copy [`examples/pr-review.yml`](examples/pr-review.yml) to `.github/workflows/pr-review.yml` in your repo.
2. Add a repo-root **`GEMINI.md`** describing what this repo's reviewer should care about
   (its rubric — e.g. the frontend/MapLibre rules for a viewer, dbt/CRS rules for a pipeline).

That's it. The caller:

```yaml
name: PR review
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  issue_comment:
    types: [created]
jobs:
  review:
    # The caller MUST grant id-token: write (it is never in the default token) or WIF auth fails.
    permissions:
      contents: read
      id-token: write
      pull-requests: write
      issues: write
    uses: UGS-GIO/ugs-gha-pr-review/.github/workflows/review.yml@v1
    secrets: inherit
```

## Who gets reviewed

- **Auto:** non-draft, non-bot PRs opened by org **members/collaborators** on a branch **in the repo** (not forks).
- **Opt-in:** a member/collaborator comments **`/review`** (or `@gemini-cli /review`) on any PR — this is how fork / outside-contributor PRs get reviewed, on demand.
- Bots, drafts, and outside contributors are **never auto-reviewed** (keeps Vertex spend gated behind a trusted human).

## Required org variables (set once, org-level)

| Variable | Value |
|---|---|
| `GCP_WIF_PROVIDER` | `projects/<num>/locations/global/workloadIdentityPools/github-actions/providers/gemini-pr-review` |
| `SERVICE_ACCOUNT_EMAIL` | `cross-model-reviewer@<project>.iam.gserviceaccount.com` |
| `GOOGLE_CLOUD_PROJECT` | the GCP project id |
| `GOOGLE_CLOUD_LOCATION` | `global` |
| `GOOGLE_GENAI_USE_VERTEXAI` | `true` |
| `GEMINI_MODEL` | e.g. `gemini-3.8-flash` (a repo may override via the `gemini_model` input) |

The reviewer service account needs `roles/aiplatform.user` and a `workloadIdentityUser` binding for
this repo's `job_workflow_ref` (`.github/workflows/review.yml@refs/tags/v1`).

## Security posture

- Every action is pinned to a commit SHA; `run-gemini-cli` at `v0.1.22`.
- `GEMINI_CLI_TRUST_WORKSPACE: false` and a read/comment-only tool set — PR diffs are treated as untrusted input.
- Least-privilege token; `persist-credentials: false` on checkout.
