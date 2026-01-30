# Copilot Instructions for Agentic Engineers

## Project Overview

This is a **GitHub Pages site** that tracks PR metrics for AI agent adoption across dotnet repos. It consists of:

1. **GitHub Actions workflow** ([.github/workflows/fetch-prs.yml](.github/workflows/fetch-prs.yml)) - Fetches PR data hourly from GitHub API
2. **Jekyll static site** ([docs/](docs/)) - Displays dashboard at https://jonathanpeppers.github.io/agentic-engineers
3. **JSON data store** ([docs/_data/prs.json](docs/_data/prs.json)) - PR data consumed by Jekyll

## Architecture & Data Flow

```
GitHub API (dotnet/* repos) → fetch-prs.yml workflow → prs.json → Jekyll → GitHub Pages
```

The workflow runs hourly (or manually with `month_year` param for backfills) and:
- Fetches merged PRs from 9 dotnet repos (maui, android, macios, etc.)
- Detects Copilot-authored PRs via author name, branch prefix (`copilot/`), labels, or body text
- Detects AI-reviewed PRs via `s/ai-agent-reviewed` label (dotnet/maui only)
- Commits updated `prs.json` back to the repo

## Key Conventions

### Workflow Script (JavaScript in YAML)
- Uses `actions/github-script@v7` with inline JavaScript
- Paginates through all PR pages until reaching `sinceDate`
- Handles concurrent runs via git stash/pop pattern (fails explicitly on merge conflicts)

### Frontend (docs/index.html)
- Single-file Jekyll page with embedded CSS/JS
- Data injected via `{{ site.data.prs | jsonify }}`
- "MAUI Reviewer" column shows `n/a` for non-maui repos (only dotnet/maui has the label)
- Repo badges use color-coded CSS classes: `.repo-android` (green), `.repo-maui` (purple), `.repo-macios` (orange)

### Adding New Repos
Add to the `repos` array in fetch-prs.yml:
```javascript
const repos = [
  { owner: 'dotnet', repo: 'new-repo' },
  // ...existing repos
];
```

### Manual Backfills
Trigger workflow manually with `month_year` parameter (format: `YYYY-MM`) to fetch historical data beyond the 90-day default window.

## Testing Changes

- Workflow changes: Create a PR - the workflow runs on `pull_request` events for its own path
- Site changes: Run Jekyll locally with `cd docs && bundle exec jekyll serve`
