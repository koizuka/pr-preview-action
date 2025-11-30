# PR Preview Action

A GitHub Action that automatically deploys pull request previews to GitHub Pages and cleans them up when the PR is closed.

When a PR is opened or updated, this action deploys a preview to `https://your-site.github.io/repo/pr/<PR_NUMBER>/` and posts a comment with the preview URL.

## Features

- **Deploy Mode**: Deploy PR preview to `gh-pages` branch under `pr/<PR_NUMBER>/`
- **Cleanup Mode**: Remove PR preview when PR is closed
- **PR Comments**: Automatic "in progress" and "deployed" comments
- **Retry Logic**: Exponential backoff for concurrent deployments
- **Deployment Wait**: Polls preview URL until available
- **Configurable**: Customizable paths, comments, timeouts

## Prerequisites

1. **GitHub Pages enabled** on your repository
   - Go to Settings > Pages
   - Source: "Deploy from a branch"
   - Branch: `gh-pages` / `/ (root)`

2. **Build output with correct base path**
   - Your build must be configured with the PR-specific base path
   - See [Build Configuration](#build-configuration) for examples

## Usage

### Deploy Preview

Add to your CI workflow (e.g., `.github/workflows/ci.yml`):

```yaml
name: CI

on:
  pull_request:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: "github-pages-deployment"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  pr-preview:
    needs: build
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - name: Deploy PR Preview
        uses: koizuka/pr-preview-action@v1
        with:
          action: deploy
          source-dir: dist
          base-url: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

### Cleanup Preview

Add a cleanup workflow (e.g., `.github/workflows/pr-cleanup.yml`):

```yaml
name: Cleanup PR Preview

on:
  pull_request:
    types: [closed]

permissions:
  contents: write

concurrency:
  group: "github-pages-deployment"
  cancel-in-progress: false

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Cleanup PR Preview
        uses: koizuka/pr-preview-action@v1
        with:
          action: cleanup
          base-url: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `action` | Action to perform: `deploy` or `cleanup` | Yes | `deploy` |
| `source-dir` | Path to built artifacts (deploy only) | No | `dist` |
| `base-url` | GitHub Pages base URL | Yes | - |
| `token` | GitHub token | Yes | `${{ github.token }}` |
| `pr-number` | PR number (auto-detected) | No | - |
| `preview-path-prefix` | Preview directory prefix | No | `pr` |
| `comment-enabled` | Enable PR comments | No | `true` |
| `wait-for-deployment` | Wait for deployment | No | `true` |
| `max-wait-time` | Max wait time (seconds) | No | `300` |
| `git-user-name` | Git user name for commits | No | `github-actions[bot]` |
| `git-user-email` | Git user email for commits | No | `github-actions[bot]@users.noreply.github.com` |

## Outputs

| Output | Description |
|--------|-------------|
| `preview-url` | The deployed preview URL |
| `has-changes` | Whether changes were deployed |

## Required Permissions

```yaml
permissions:
  contents: write      # For pushing to gh-pages
  pull-requests: write # For PR comments (deploy only)
```

## How It Works

### Deploy Mode

1. Validates inputs and determines PR number
2. Checks for existing preview comment to avoid duplicates
3. Creates "in progress" comment on PR
4. Clones or creates `gh-pages` branch
5. Creates `pr/<PR_NUMBER>/` directory and copies build artifacts
6. Commits and pushes with retry logic (exponential backoff)
7. Waits for GitHub Pages deployment
8. Updates PR comment with preview URL

### Cleanup Mode

1. Validates inputs and determines PR number
2. Clones `gh-pages` branch (exits gracefully if not exists)
3. Removes `pr/<PR_NUMBER>/` directory
4. Removes `pr/` directory if empty
5. Commits and pushes changes

## Directory Structure on gh-pages

```
gh-pages/
├── index.html (main site)
├── [other main site files]
└── pr/
    ├── 1/
    │   └── [PR #1 preview files]
    ├── 2/
    │   └── [PR #2 preview files]
    └── 3/
        └── [PR #3 preview files]
```

## Concurrency

Use a shared concurrency group to prevent race conditions:

```yaml
concurrency:
  group: "github-pages-deployment"
  cancel-in-progress: false
```

## Build Configuration

Your build tool must output assets with the correct base path for PR previews.

### Vite

Create a dynamic config in your workflow:

```yaml
- name: Build with PR-specific base path
  run: |
    cat > vite.config.pr.ts << 'EOF'
    import { defineConfig } from 'vite'
    import react from '@vitejs/plugin-react'

    export default defineConfig({
      plugins: [react()],
      base: '/your-repo/pr/${{ github.event.pull_request.number }}/',
    })
    EOF

    npx vite build --config vite.config.pr.ts
```

### Webpack

```yaml
- name: Build with PR-specific base path
  env:
    BASE_PATH: /your-repo/pr/${{ github.event.pull_request.number }}/
  run: |
    cat > webpack.pr.config.js << 'EOF'
    import baseConfig from './webpack.config.js';

    export default {
      ...baseConfig,
      output: {
        ...baseConfig.output,
        publicPath: process.env.BASE_PATH || '/',
      },
    };
    EOF

    NODE_ENV=production npx webpack --config webpack.pr.config.js
```

### Next.js

```yaml
- name: Build with PR-specific base path
  run: |
    npx next build
    npx next export -o out
  env:
    NEXT_PUBLIC_BASE_PATH: /your-repo/pr/${{ github.event.pull_request.number }}
```

## Troubleshooting

### Preview URL returns 404

- Ensure GitHub Pages is enabled and set to deploy from `gh-pages` branch
- Check that your build output has the correct base path
- Wait a few minutes for GitHub Pages to deploy (this action waits up to 5 minutes by default)

### Push conflicts

This action uses exponential backoff retry (up to 5 attempts) to handle concurrent deployments. If you still see conflicts:
- Ensure all workflows use the same concurrency group
- Set `cancel-in-progress: false` to prevent cancellation during deployment

### Comment not appearing

- Ensure `pull-requests: write` permission is set
- Check if `comment-enabled` input is set to `true` (default)

## Limitations

- Only works with GitHub Pages deployed from `gh-pages` branch
- Requires build artifacts to be pre-built with the correct base path
- PR previews are stored in `pr/<number>/` subdirectory (not customizable path structure)

## License

MIT
