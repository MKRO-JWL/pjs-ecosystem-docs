# PJS Ecosystem Docs

Single source of truth (SSOT) documentation for the PJS Collectables ecosystem. The canonical document is `SOFTWARE_ECOSYSTEM.md`, and this repository is responsible for broadcasting updates to peer repositories when that file changes.

## What lives here

- `SOFTWARE_ECOSYSTEM.md`: The canonical ecosystem documentation.
- `.github/workflows/dispatch_to_peers.yml`: Sends a `repository_dispatch` event to each registered peer repo whenever `SOFTWARE_ECOSYSTEM.md` changes on `main`.

## How other projects receive the document

To attach another project so it **receives** the latest `SOFTWARE_ECOSYSTEM.md`, add a workflow in the target repository that listens for the `sync_requested` dispatch event and then pulls the file from this repo.

### 1) Add a workflow in the target repository

Create `.github/workflows/sync_ecosystem_docs.yml` (or similar) with the following example (update the root branch if needed):

```yaml
name: Sync SOFTWARE_ECOSYSTEM.md


#NOTICE: If the root branch differs make sure to update it to either master/main or any other used root.

on:
  push:
    # Always update the SSOT repo on push
    branches: [master]
  repository_dispatch:
    # Trigger action to fetch newest md from SSOT repo
    types: [sync_requested]

env:
  PUBLIC_MD_URL: https://raw.githubusercontent.com/MKRO-JWL/pjs-ecosystem-docs/main/SOFTWARE_ECOSYSTEM.md
  PUBLIC_MD_REPO: MKRO-JWL/pjs-ecosystem-docs

jobs:
  sync-md:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      # === Push-triggered block ===
      - name: Push to SSOT
        if: github.event_name == 'push' && github.actor != 'github-actions[bot]'
        run: |
          echo "Pushing to public MD repo..."
          git clone https://x-access-token:${{ secrets.PUBLIC_MD_PUSH_TOKEN }}@github.com/$PUBLIC_MD_REPO temp_md_repo
          cp SOFTWARE_ECOSYSTEM.md temp_md_repo/SOFTWARE_ECOSYSTEM.md
          cd temp_md_repo
          git config user.name "GitHub Action"
          git config user.email "action@github.com"
          git add SOFTWARE_ECOSYSTEM.md
          git diff --cached --quiet && echo "No changes." || git commit -m "Sync from ${{ github.repository }}"
          git push origin main

      # === Dispatch-triggered block ===
      - name: Fetch from SSOT
        if: github.event_name == 'repository_dispatch'
        run: |
          echo "Fetching from SSOT..."
          curl -s -f -o SOFTWARE_ECOSYSTEM.md "$PUBLIC_MD_URL"

      - name: Commit if changed
        if: github.event_name == 'repository_dispatch'
        run: |
          git config user.name "GitHub Action"
          git config user.email "action@github.com"
          git add SOFTWARE_ECOSYSTEM.md
          
          if ! git ls-files --error-unmatch SOFTWARE_ECOSYSTEM.md >/dev/null 2>&1; then
            echo "New file not tracked. Committing."
            git commit -m "Add SSOT file" SOFTWARE_ECOSYSTEM.md
            git push origin HEAD
          elif git diff --cached --quiet; then
            echo "No changes from SSOT."
          else
            echo "Updating local file."
            git commit -m "Sync from SSOT"
            git push origin HEAD
          fi
```

### 2) Ensure the target repo can accept dispatch events

The target repository **does not need** any extra secret for receiving `repository_dispatch` events. It just needs the workflow above committed to the default branch. 

**Important: Repo Settings → Actions → Generel → Workflow permissions MUST be on Read and Write Permssion!**

### 3) Optional: protect the doc location

If your repo uses branch protections, ensure that your workflow is allowed to push commits.

## How another Repository starts pushing updates to the SSOT doc

When a repo wants to **push its local `SOFTWARE_ECOSYSTEM.md`** back to the SSOT:

1. Add the workflow above to that repo (the `push` trigger handles the upload).
2. Create a secret named `PUBLIC_MD_PUSH_TOKEN` in that repo.
3. Ensure the token has access to `MKRO-JWL/pjs-ecosystem-docs` with write permission.
4. Update the workflow’s `branches: [master]` if the repo uses a different default branch.

This is all that’s required for pushing; no dispatch token is needed in the sender repo.

## What this Repository needs to include a new project

When you want this repo to **notify a new project**:

1. **Add the repo to the dispatch list** in `.github/workflows/dispatch_to_peers.yml` under the `REPOLIST` heredoc.
2. **Ensure `DISPATCH_PAT` is set** in this repo’s secrets. It must have permission to call `repository_dispatch` on the target repo(s). See Steps below.
3. **Verify the target repo has the receiver workflow** (described above) so it can act on `sync_requested`. Make sure any recipient Repository is merged to main branch to avoid conflicts!

## Token setup and where to edit fine-grained access

The workflows use two tokens:

- `PUBLIC_MD_PUSH_TOKEN`: token used to push updates to `https://github.com/MKRO-JWL/pjs-ecosystem-docs.git` from other repositories.
- `DISPATCH_PAT`: token used in this repo to dispatch `sync_requested` events to all registered repos.

In the current setup, the tokens are stored in the `Desktop/Business/Server` folder.

To edit fine-grained token access in GitHub:

1. Go to **GitHub** → **Settings** → **Developer settings**.
2. Open **Personal access tokens** → **Fine-grained tokens**.
3. Select the token you want to edit (e.g., `PUBLIC_MD_PUSH_TOKEN` or `DISPATCH_PAT`).
4. Click **Edit**, then adjust repository access and permissions.

## Manual sync (local)

If you need to update the doc without the automation, you can copy `SOFTWARE_ECOSYSTEM.md` into the target repo and commit it directly. The automation is just a convenience to keep the file in sync.
