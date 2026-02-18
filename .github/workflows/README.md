# Workflow Documentation

## Publish Release Assets

This workflow automatically publishes release assets from the `singlestore/language-server` repository to this repository (`singlestore-labs/language-server`).

### How It Works

The workflow:
1. Fetches release information from the source repository
2. Downloads all release assets (excluding source code archives)
3. Creates a new release in this repository with the same tag, name, and description
4. Uploads the downloaded assets to the new release

### Triggering the Workflow

#### Manual Trigger (workflow_dispatch)

You can manually trigger the workflow from the GitHub Actions UI:
1. Go to Actions tab
2. Select "Publish Release Assets" workflow
3. Click "Run workflow"
4. Enter the tag name from the source repository (e.g., `v1.0.0`)

#### Automatic Trigger (repository_dispatch)

The workflow can be triggered automatically using the GitHub API or from another workflow:

```bash
curl -X POST \
  -H "Accept: application/vnd.github.v3+json" \
  -H "Authorization: token YOUR_TOKEN" \
  https://api.github.com/repos/singlestore-labs/language-server/dispatches \
  -d '{"event_type":"new-release","client_payload":{"tag_name":"v1.0.0"}}'
```

### Required Secrets

The workflow requires the following secret to be configured in the repository settings:

- `SOURCE_REPO_TOKEN`: A GitHub Personal Access Token (PAT) with `repo` scope that has read access to the `singlestore/language-server` repository

To add this secret:
1. Go to repository Settings > Secrets and variables > Actions
2. Click "New repository secret"
3. Name: `SOURCE_REPO_TOKEN`
4. Value: Your GitHub PAT with read access to the source repository

### Asset Filtering

The workflow automatically excludes GitHub's auto-generated source code archives:
- "Source code (zip)"
- "Source code (tar.gz)"

All other assets (binaries, packages, etc.) will be downloaded and re-published.

### Permissions

The workflow requires `contents: write` permission to create releases and push tags.
