---
title: "Mirror Azure DevOPS Repository to Github"
date: 2025-10-21
published: true
categories:
  - blog
tags:
  - Github Page
  - Azure DevOPS
---

# Mirroring an Azure DevOps Repo to GitHub with Azure Pipelines

Keeping Azure DevOps (AzDO) and GitHub repositories in sync makes sense when you want Azure DevOPS Pipelines orchestration but also need GitHub for publishing, Pages, or collaboration. This post walks through a secure, automated mirror from AzDO → GitHub using IaC-friendly steps.

---

## Prerequisites

- Azure DevOps project with an existing Git repository
- GitHub repository (target mirror)
- Azure DevOps PAT (for pipeline tasks, automatically provided)
- GitHub Personal Access Token (PAT) with **repo** scope
- Pipeline contributor rights and ability to create variable groups
- `.gitignore` updated so only intended files are tracked

---

## Step 1 – In GitHub, Create a GitHub PAT

1. In GitHub, go to **Settings → Developer settings → Personal access tokens → Fine-grained or Classic**.
2. Generate a token with the `repo` scope (minimum: Contents: read/write).
3. Copy the token to the Azure DevOPS Project's Pipeline Library, storing encrypted in a variable group named `gh-vars`.

---

## Step 2 – In Azure DevOPS, Store Credentials in Azure DevOps Project Pipeline Library

1. Navigate to **Azure DevOps → Pipelines → Library → Variable groups → + Variable group**.
2. Name it (e.g., `gh-vars`) and add the following variables:

   | Variable        | Example Value                            | Secret? |
   |-----------------|------------------------------------------|---------|
   | `GITHUB_OWNER`  | `<Your GitHub username>`                 | No      |
   | `GITHUB_REPO`   | `<Your repository name>`                 | No      |
   | `GITHUB_TOKEN`  | `<GitHub PAT>`                           | Yes     |
   | `GITHUB_NAME`   | `<Your git name>`                        | No      |
   | `GITHUB_REMOTE_URL`   | `</your-github-username/your-repository-name>` | No      |
   | `GITHUB_USER`   | `<Your git email>`                       | No      |
   

3. Save the variable group and allow the pipeline to access it.

---

## Step 3 – Create the Mirror Pipeline

Create `azure-pipelines-sync.yml` (or reuse an existing pipeline). Key components:

```yaml
trigger:
  branches:
    include:
      - main       # mirror whenever main changes

pool:
  vmImage: ubuntu-latest

variables:
- group: gh-vars   # import the GitHub variables

steps:
- checkout: self
  clean: true
  fetchDepth: 0
  persistCredentials: true

- script: |
    set -euo pipefail

    git config user.name "$(GITHUB_NAME)"
    git config user.email "$(GITHUB_USER)"

    # Reset repo to Azure DevOps state
    git fetch origin
    git checkout "$(Build.SourceBranchName)"
    git reset --hard "origin/$(Build.SourceBranchName)"

    # Configure GitHub remote
    if git remote get-url github >/dev/null 2>&1; then
      git remote remove github
    fi

    git remote add github "https://x-access-token:${GITHUB_TOKEN}@github.com/${GITHUB_OWNER}/${GITHUB_REPO}.git"

    # Optional: fetch GitHub for visibility before overwriting
    git fetch github "$(Build.SourceBranchName)" || true

    # Mirror Azure DevOps to GitHub (overwrites remote history)
    git push github "$(Build.SourceBranchName)" --force-with-lease
  displayName: Mirror to GitHub
  env:
    GITHUB_TOKEN: $(GITHUB_TOKEN)
    GITHUB_OWNER: $(GITHUB_OWNER)
    GITHUB_REPO: $(GITHUB_REPO)
```
### Notes

- fetchDepth: 0 ensures full commit history is available for pushes.
- persistCredentials: true keeps the system access token so fetch/reset from AzDO works without prompts.
- --force-with-lease overwrites GitHub only if its state hasn’t changed unexpectedly; use --force if you must clobber regardless (less safe).

## Step 4 – Queue the Pipeline & Authorize

1. In Azure DevOps → Pipelines → New pipeline → Select the repo → Existing Azure Pipelines YAML file → pick the sync YAML.
2. Authorize the pipeline to use the GitHub variable group if prompted.
3. Queue a run (or commit to main).
4. Inspect logs—look for git push github without errors.

## Step 5 – Validate the Mirror

- Check the GitHub repository to ensure commits match Azure DevOps.
- Enable GitHub branch protection (optional) to prevent manual edits.
- Configure notifications or dashboards to monitor pipeline success.

## Step 6 – Keep Repos Clean

- .gitignore only blocks new files. If sensitive assets were committed previously, remove them from Git history:
```bash
git rm --cached path/to/file
git commit -m "Remove tracked file"
git push
```
- Once removed from Azure DevOps, the mirror will also drop them on GitHub.

## Security & Governance Checklist

- Rotate GitHub PAT regularly; store in variable group or Key Vault.
- Limit pipeline permissions to minimal required scopes/branches.
- Review pipeline logs for credential exposure; avoid set -x.
- Never hard code access keys, PAT tokens or passwords. Always use Library encrypted variables or Azure Key Vault.
- Document ownership of the sync pipeline for audit compliance.

With these steps, Azure DevOps becomes your source of truth while GitHub receives an up-to-date mirror automatically—perfect for GitHub Pages, public visibility, or cross-platform workflows without manual copying.

Happy mirroring! 🚀