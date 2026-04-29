# PR Preview Action

GitHub Composite Action for deploying PR previews to gh-pages branch.

For the current version, run `gh release list` or check
<https://github.com/koizuka/pr-preview-action/releases>.

## Project Structure

```text
.
├── action.yml              # Main composite action definition
├── README.md               # User documentation
├── CLAUDE.md               # Project notes for Claude Code
├── LICENSE                 # MIT License
├── .yamllint               # yamllint config (YAML style)
├── .markdownlint.json      # markdownlint-cli2 config
└── .github/
    ├── dependabot.yml      # Dependabot config for action updates
    └── workflows/
        ├── ci.yml          # Lint & validate (actionlint, shellcheck, yamllint, markdownlint, schema, hash-pinning)
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
- Action versions: Pin by commit hash with version comment (e.g., `@abc123 # v1.0.0`).
  Enforced by the `hash-pin-check` job in CI.

## CI Checks

The `ci.yml` workflow runs six parallel jobs:

- `actionlint` — workflow lint with shellcheck integration
- `action-schema` — `mpalmer/action-validator` against the official action schema
- `shellcheck-action` — extracts each `shell: bash` step from `action.yml` and runs shellcheck
- `yamllint` — YAML style for `action.yml` and workflows
- `markdownlint` — Markdown lint for `**/*.md`
- `hash-pin-check` — enforces 40-char SHA pinning on every `uses:` reference

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
