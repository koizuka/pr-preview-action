# PR Preview Action

GitHub Composite Action for deploying PR previews to gh-pages branch.

## Current Version

- **v1.0.0**: `e7366d355cc0c6a1b7d96965f1d3c82a82287297`

## Project Structure

```
.
├── action.yml              # Main composite action definition
├── README.md               # User documentation
├── LICENSE                 # MIT License
└── .github/
    ├── dependabot.yml      # Dependabot config for action updates
    └── workflows/
        ├── ci.yml          # Lint and validate action.yml
        └── release.yml     # Auto-update major version tag on release
```

## How It Works

- **Deploy mode** (`action: deploy`): Creates `pr/<PR_NUMBER>/` directory on gh-pages branch and copies build artifacts
- **Cleanup mode** (`action: cleanup`): Removes the PR preview directory when PR is closed

## Key Features

- PR comment management with duplicate prevention (uses `<!-- PR Preview Comment -->` marker)
- Exponential backoff retry for concurrent deployments (max 5 retries)
- Auto-create gh-pages branch if not exists
- Polls preview URL until available (configurable timeout)

## Development

### Testing Changes

This action cannot be easily unit tested. Test by:
1. Creating a test repository
2. Using the action from a branch: `uses: koizuka/pr-preview-action@branch-name`

### Releasing

```bash
gh release create v1.x.x --generate-notes
```

The release workflow automatically updates the `v1` major tag.

### Dependencies

Uses `actions/github-script` for PR comment management. Version pinned by commit hash for security.

## Conventions

- Commit messages: Use conventional commits (e.g., `chore:`, `fix:`, `feat:`)
- Action versions: Pin by commit hash with version comment (e.g., `@abc123 # v1.0.0`)

## Usage Example

```yaml
# Deploy (in ci.yml)
- uses: koizuka/pr-preview-action@e7366d355cc0c6a1b7d96965f1d3c82a82287297 # v1.0.0
  with:
    action: deploy
    source-dir: dist
    base-url: https://owner.github.io/repo
    token: ${{ secrets.GITHUB_TOKEN }}

# Cleanup (in pr-cleanup.yml)
- uses: koizuka/pr-preview-action@e7366d355cc0c6a1b7d96965f1d3c82a82287297 # v1.0.0
  with:
    action: cleanup
    base-url: https://owner.github.io/repo
    token: ${{ secrets.GITHUB_TOKEN }}
```
