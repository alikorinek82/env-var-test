# GitHub Actions — Cross-repo Access via GitHub App

A workflow running in `env-var-test` generates a scoped token via a GitHub App, checks out `aktestak`, copies a file, and opens a Pull Request — all without a PAT or hardcoded credentials.

---

## How it works

```
env-var-test (workflow runs)
    │
    ├─ 1. Request token ──► GitHub App ──► token scoped to aktestak only
    │
    ├─ 2. Checkout aktestak using token
    │
    ├─ 3. Copy shared/config.yml → aktestak/shared/config.yml
    │
    └─ 4. Open PR in aktestak  ──► PR opened as github-app[bot]
```

Key principle: the token is scoped to `aktestak` only. Even though secrets live in `env-var-test`, the token cannot write anywhere else.

---

## Prerequisites

- GitHub App created and installed on **both** repositories
- Two secrets added to `env-var-test` → Settings → Secrets and variables → Actions:

| Secret name | Value |
|---|---|
| `CLIENT_ID` | The App ID number from your GitHub App's detail page |
| `APP_PRIVATE_KEY` | Full contents of the `.pem` private key file |

> **Where to find these:** GitHub → Settings → Developer settings → GitHub Apps → your App → "About" section for App ID, "Private keys" section to generate and download the `.pem` file.

No secrets need to be added to `aktestak`.

---

## Workflow file — env-var-test

Save as `.github/workflows/sync-to-aktestak.yml` in `env-var-test`:

```yaml
name: Sync file to aktestak

on:
  push:
    branches: [ main ]
    paths:
      - '.github/shared/config.yml'   # only triggers when this file changes

jobs:
  sync:
    runs-on: ubuntu-latest

    steps:
      # 1. Checkout env-var-test so the workflow has access to the file
      - name: Checkout env-var-test
        uses: actions/checkout@v4

      # 2. Generate a scoped token via GitHub App
      - name: Get App token
        uses: actions/create-github-app-token@v1
        id: app-token
        with:
          app-id:      ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          repositories: aktestak    # token is valid for this repo only

      # 3. Checkout aktestak using the App token
      - name: Checkout aktestak
        uses: actions/checkout@v4
        with:
          repository: alikorinek82/aktestak
          token:      ${{ steps.app-token.outputs.token }}
          path:       aktestak      # checks out into ./aktestak folder

      # 4. Copy the file from alpha into beta
      - name: Copy config
        run: |
          mkdir -p aktestak/shared
          cp .github/shared/config.yml aktestak/shared/config.yml

      # 5. Open a Pull Request in aktestak
      - name: Open Pull Request in aktestak
        uses: peter-evans/create-pull-request@v6
        with:
          token:          ${{ steps.app-token.outputs.token }}
          path:           aktestak
          branch:         sync/config-from-alpha
          title:          "sync: config from env-var-test (${{ github.sha }})"
          body:           "Automated sync of shared/config.yml"
          commit-message: "chore: sync config from env-var-test"
```

---

## Optional — auto-merge in aktestak

If you want sync PRs to merge automatically without manual approval, add this workflow to `aktestak`:

```yaml
name: Auto-merge sync PR

on:
  pull_request:
    branches: [ main ]

jobs:
  automerge:
    if: startsWith(github.head_ref, 'sync/')
    runs-on: ubuntu-latest
    steps:
      - name: Auto-merge sync PR
        run: gh pr merge --auto --squash "${{ github.event.pull_request.number }}"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}   # standard GITHUB_TOKEN is enough here
```

> **Note:** Auto-merge only works if it is enabled in `aktestak` settings: Settings → General → Allow auto-merge.

---

## What you see after a successful run

In `env-var-test` Actions tab:

```
✓  Checkout env-var-test                2s
✓  Get App token                      1s
     Generated token for installation 12345001
     Token expires: 2026-04-21T15:42:00Z
✓  Checkout aktestak                 3s
     Cloning into 'aktestak'...
✓  Copy config                        <1s
✓  Open Pull Request in aktestak     4s
     Created pull request #42
     https://github.com/your-username/aktestak/pull/42
```

The PR in `aktestak` will appear as opened by **github-app[bot]**, not your personal account. This is the correct and expected behaviour.

---

## Further steps

- **Filter by file paths** — extend the `paths:` trigger to watch multiple files
- **Local testing** — use [`act`](https://github.com/nektos/act) to run the workflow locally before pushing
- **Slack notification** — add a step after the PR creation to post a message to Slack if the sync fails
