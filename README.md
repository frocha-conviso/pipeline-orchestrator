# Conviso AST — Pipeline Orchestrator

Template pipeline that runs [Conviso AST](https://docs.convisoappsec.com) against your
repositories. You keep **one** orchestrator project; the Conviso Platform triggers it and
tells it which repository and branch to scan. Nothing is added to the repositories
themselves.

Two flows use the same pipeline:

| Flow | Trigger | Extra inputs |
|---|---|---|
| Post-merge scan | A pull/merge request is merged | `commit_sha`, `pr_number` |
| Run AST (on demand) | Someone clicks **Run AST** on an asset in the platform | `scan_run_id` |

## Setup

**1. Copy this repository** into your organization. Keep only the file for your provider:

| Provider | Keep this file |
|---|---|
| GitHub | `.github/workflows/ast.yml` |
| GitLab | `.gitlab-ci.yml` |
| Azure DevOps | `azure-pipelines.yml` |
| Bitbucket Cloud | `bitbucket-pipelines.yml` |

**2. Configure two values** in the orchestrator project:

| Name | Kind | What it is |
|---|---|---|
| `CONVISO_API_KEY` | Secret | Your Conviso Platform API key |
| `CONVISO_COMPANY_ID` | Variable | Your Conviso company id |

- **GitHub** — both under *Settings > Secrets and variables > Actions* (`CONVISO_API_KEY` as a secret, `CONVISO_COMPANY_ID` as a variable).
- **GitLab / Azure DevOps / Bitbucket** — both as pipeline/repository variables; only the API key marked as masked/secured.

No personal access token is needed.

The company id is a constant for this project: one orchestrator serves one integration, and
an integration belongs to one company. Post-merge scans do not carry it — the platform only
sends it on the **Run AST** flow — so the pipeline falls back to this value. When the
platform does send one, the sent value wins.

The credential each CI provider injects by default (`GITHUB_TOKEN`, `CI_JOB_TOKEN`, the build
service account, Bitbucket OAuth for the orchestrator repo) only reaches the orchestrator
project itself, so it cannot clone the repository being scanned. Instead of asking you to
store a long-lived token for that, the pipeline asks the Conviso Platform for one at the
start of every run, using the API key it already has. The platform only issues it for
repositories your company has imported as assets.

On GitHub that credential is a GitHub App installation token restricted to the **single
repository being scanned**, with **read-only** access to its contents, valid for about an
hour. On GitLab, Azure DevOps and Bitbucket the providers offer no per-repository
credential of that kind, so the integration's own token is used — still issued per run and
revocable from the platform, but broader in scope.

**3. Point the platform at the orchestrator** — in the platform, open your integration and
fill in the Orchestrator Configuration:

- **GitHub** — repository (`owner/repo`), workflow file (`ast.yml`), ref
- **GitLab** — project id, pipeline ref
- **Azure DevOps** — organization, project, pipeline id, ref
- **Bitbucket** — workspace, repository, ref (custom pipeline name must be `run-ast-scan`)

**4. Associate your repositories** in the platform so each one becomes an asset. The
**Run AST** button appears on repository assets once the steps above are done.

## Inputs

Every provider receives the same inputs, in this order (matches the Platform dispatcher):

| Input | Environment variable | Notes |
|---|---|---|
| `repo_full_name` | `CONVISO_REPO_FULL_NAME` | Repository to scan (`owner/repo` or GitLab path). Required |
| `branch` | `CONVISO_BRANCH` | Branch to scan. Required |
| `commit_sha` | — | Post-merge only |
| `pr_number` | — | Post-merge only. Wire renames: Bitbucket `pr_id`, GitLab `mr_iid` |
| `api_url` | `CONVISO_BASE_URL` | Defaults to `https://api.convisoappsec.com` |
| `company_id` | `CONVISO_COMPANY_ID` | Sent by **Run AST**; otherwise the project variable is used |
| `asset_id` | `CONVISO_ASSET_ID` | Empty means resolve by repository name |
| `scan_run_id` | `CONVISO_SCAN_RUN_ID` | Links this execution to the run in the platform |

`CONVISO_APIKEY`, `CONVISO_BASE_URL` and `CONVISO_COMPANY_ID` are required by the scanner.
The secret you create is named `CONVISO_API_KEY`; the pipeline maps it to `CONVISO_APIKEY`.

Providers differ only where the CI forces it: `--provider`, clone URL/auth, and YAML syntax.
Azure also needs an empty container entrypoint and publishes artifacts from the host.

## How the clone credential is obtained

The scanner image ships `conviso-ast-repository-token`, which prints a credential for the
repository under scan:

```bash
conviso-ast-repository-token --provider github         # or gitlab, azure_devops, bitbucket
```

The pipeline calls it before checkout and hands the result to the clone step. It prints
**only** the token on stdout — errors go to stderr and exit non-zero.

## Keeping it current

The pipeline runs `convisoappsec/convisoast_v2:latest`, so it picks up scanner releases on
its own. The **input list** does not: if you pin an older copy of this template, new inputs
the platform sends are dropped and the scan still runs, but the platform loses track of that
execution and marks the run as lost after a while. Re-copy this file when you notice runs
finishing without appearing in the UI.
