# Copilot Instructions for Agentic Engineers

## Overview

GitHub Pages site tracking PR metrics for AI agent adoption across dotnet repos.

**Components:** Workflow ([fetch-prs.yml](.github/workflows/fetch-prs.yml)) → [prs.json](docs/_data/prs.json) → Jekyll ([docs/](docs/))

## Workflow Behavior

Runs hourly (or manually with `month_year` param like `2025-06` for backfills):
- Fetches merged PRs from 9 dotnet repos (maui, android, macios, etc.)
- Detects Copilot PRs via author, `copilot/` branch prefix, labels, or body
- Detects AI-reviewed PRs via `s/ai-agent-reviewed` label (dotnet/maui only)
- Uses git stash/pop for concurrent runs (fails on merge conflicts)

## Frontend (docs/index.html)

Single-file Jekyll page. Data via `{{ site.data.prs | jsonify }}`. "MAUI Reviewer" shows `n/a` for non-maui repos. Repo badges: `.repo-android` (green), `.repo-maui` (purple), `.repo-macios` (orange).

## Adding Repos

Add to `repos` array in fetch-prs.yml: `{ owner: 'dotnet', repo: 'new-repo' }`

## Testing

- Workflow: Create PR (auto-runs on workflow file changes)
- Site: `cd docs && bundle exec jekyll serve`
