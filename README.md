# Changelog Action

GitHub Action to automate release management and website changelog updates for SQLite Cloud.

## What this Action does

This action automates the entire release process when a new tag is created in your repository:

1. **Creates a GitHub Release**: Automatically extracts the changelog from `CHANGELOG.md` for the current version and creates a GitHub release
2. **Updates the website**: Extracts all changelog entries **except** those marked as `**[PRIVATE]**` and creates a JSON file in the website repository
3. **Creates a Pull Request**: Automatically opens a PR on the website repository with the new changelog file
4. **Closes resolved issues**: Identifies and automatically closes all issues mentioned in the changelog (format `[#123]`), adding a comment with a link to the release

## How to use this Action

### 1. Create the workflow in your repository

In the repository where you want to use this action, create the file `.github/workflows/release.yml`:

```yaml
name: Release and Update Website Changelog

on:
  push:
    tags:
      - "v*.*.*"
  workflow_dispatch:
    inputs:
      test_version:
        description: "The fake version to test (e.g., 99.9.9)"
        required: true
        default: "99.9.9"

jobs:
  release:
    # Pin to a reviewed release tag (or, strongest, a full commit SHA) — not @main. See "Security" below.
    uses: sqlitecloud/changelog-action/.github/workflows/action.yml@v1
    with:
      name: "Dashboard" # Replace with your system name (e.g., "API", "SDK", "Mobile App", etc.)
      stage_branch: ${{ vars.WEBSITE_STAGE_BRANCH }} # staging branch changelog PRs target
      # base_branch: main                            # optional, defaults to "main"
    secrets:
      personal_access_token: ${{ secrets.WEBSITE_REPO_PAT }}
      website_repo: ${{ secrets.WEBSITE_REPO }}
      website_changelog_dir: ${{ secrets.WEBSITE_CHANGELOG_DIR }}
```

**Note**: Customize the `name` parameter with your system/product name. `stage_branch` is required; `base_branch` defaults to `main`. Passing the staging branch via a repo/org **variable** (`vars.*`) keeps the internal branch name out of your workflow file too, though branch names are far less sensitive than the repo/path.

**Recommended: pass secrets explicitly** (as shown above). Listing each secret by name means only these three values are handed to the workflow — nothing else the caller has access to. This is the safest option, and it is required when the caller lives in a different organization than this action.

<details>
<summary>Alternative: <code>secrets: inherit</code></summary>

If the caller repo (or its organization) defines secrets whose names **exactly match** the ones this workflow declares — `personal_access_token`, `website_repo`, `website_changelog_dir` — you can replace the whole `secrets:` mapping with a single line:

```yaml
    secrets: inherit
```

This forwards **all** secrets the caller has access to into the reusable workflow, which then uses the ones it declares by name. Note that secret names are case-insensitive (so `WEBSITE_REPO` matches `website_repo`), but the token secret must be named `personal_access_token`, not `WEBSITE_REPO_PAT`, for `inherit` to satisfy it. Prefer the explicit form unless you own both repos and want new secrets to flow through automatically.

</details>

#### Skip GitHub Release creation

If you only want to update the website changelog without creating a GitHub Release (e.g., the release is managed separately), set `create_release: false`:

```yaml
jobs:
  release:
    uses: sqlitecloud/changelog-action/.github/workflows/action.yml@v1
    with:
      name: "Dashboard"
      create_release: false
      stage_branch: ${{ vars.WEBSITE_STAGE_BRANCH }}
    secrets:
      personal_access_token: ${{ secrets.WEBSITE_REPO_PAT }}
      website_repo: ${{ secrets.WEBSITE_REPO }}
      website_changelog_dir: ${{ secrets.WEBSITE_CHANGELOG_DIR }}
```

The default is `true`, so omitting the parameter keeps the existing behavior.

### 2. Configure the GitHub secrets

Go to `Settings` → `Secrets and variables` → `Actions` (at the repository or, preferably, the **organization** level so all repos share them) and add the following secrets:

| Secret | Value |
| ------ | ----- |
| `WEBSITE_REPO_PAT` | A GitHub Personal Access Token with write permissions to the website repository |
| `WEBSITE_REPO` | The target website repository in `owner/repo` form |
| `WEBSITE_CHANGELOG_DIR` | The repo-relative directory where changelog JSON files are written (e.g. `path/to/changelogs`) |

`WEBSITE_REPO` and `WEBSITE_CHANGELOG_DIR` are passed as secrets (rather than hardcoded in the workflow) so this repository can be public without exposing the target repo name or path. Their values are also masked in the run logs.

The examples also read the staging branch from a **variable** rather than a secret:

| Variable | Value |
| -------- | ----- |
| `WEBSITE_STAGE_BRANCH` | The staging branch on the website repo (passed to the `stage_branch` input) |

A variable (not a secret) is appropriate here because a branch name is low-sensitivity and, unlike a secret, it is **not** masked in logs — which keeps the promotion steps readable. You can also just hardcode `stage_branch:` directly in your caller workflow if you don't mind the branch name being visible there (your caller repo can be private).

### 3. CHANGELOG.md format

The `CHANGELOG.md` file must follow this format (Format based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)):

```markdown
## [1.2.3] - 2026-01-01

### Added

- [#1000](https://github.com/your-org/your-repo/issues/1000) **Advanced Query Builder**: Introduced a visual query builder interface for creating complex SQL queries without writing code.
  - Added drag-and-drop table joining interface with automatic relationship detection.
  - Implemented smart autocomplete for column names and SQL functions.
  - Created query template library with common patterns and best practices.
  - Added query validation with real-time syntax checking and suggestions.
- **[PRIVATE]** [#1001](https://github.com/your-org/your-repo/issues/1001) Implemented internal caching layer for frequently accessed metadata.
- **[PRIVATE]** [#1002](https://github.com/your-org/your-repo/issues/1002) Added telemetry collection for performance monitoring.
- **Dark Mode Support**: Full dark theme implementation across the entire dashboard.
  - Automatic theme detection based on system preferences.
  - Manual theme toggle with persistent user preference.
- **[PRIVATE]** [#1003](https://github.com/your-org/your-repo/issues/1003) Enhanced logging system for debugging edge function executions.

### Changed

### Fixed

- **Connection Stability Improvements**: Resolved intermittent connection drops affecting query execution in high-traffic scenarios.
  - Fixed WebSocket reconnection logic to handle network interruptions gracefully.
  - Improved connection pooling to prevent resource exhaustion.
  - Added automatic retry mechanism with exponential backoff for failed queries.
- **[PRIVATE]** [#1004](https://github.com/your-org/your-repo/issues/1004) Fixed memory leak in the database connection manager.
- **[PRIVATE]** [#1005](https://github.com/your-org/your-repo/issues/1005) Corrected timezone handling in scheduled task execution.
- **Query Result Export**: Fixed issue where large CSV exports would timeout or corrupt data.
  - Implemented chunked export processing for datasets exceeding 50,000 rows.
  - Fixed character encoding issues with international characters in CSV format.
- **[PRIVATE]** [#1006](https://github.com/your-org/your-repo/issues/1006) Resolved race condition in billing operation processing.

### Security

- **Enhanced Authentication Security**: Strengthened authentication mechanisms to protect against unauthorized access.
  - Implemented rate limiting on login endpoints to prevent brute-force attacks (max 5 attempts per 15 minutes).
  - Added support for hardware security keys (WebAuthn/FIDO2) as a second factor.
  - Enhanced session management with automatic invalidation of concurrent sessions from different locations.
- **[PRIVATE]** [#1007](https://github.com/your-org/your-repo/issues/1007) Upgraded JWT library to patch a token verification vulnerability.
- **[PRIVATE]** [#1008](https://github.com/your-org/your-repo/issues/1008) Implemented additional input sanitization in edge function parameters.
- **SQL Injection Prevention**: Enhanced query parameter validation to prevent SQL injection attacks.
  - Added prepared statement enforcement for all user-generated queries.
  - Implemented automatic detection and blocking of suspicious query patterns.
- **[PRIVATE]** [#1009](https://github.com/your-org/your-repo/issues/1009) Fixed XSS vulnerability in webhook payload display.

### Removed
```

**Important**:

- **All entries are published on the website by default**, except those marked with `**[PRIVATE]**`
- Entries marked with `**[PRIVATE]**` are excluded from the website changelog (internal changes only)
- The `**[PUBLIC]**` tag is still supported for backwards compatibility, but no longer necessary
- Issues must be referenced with the format `[#number](url)` to be automatically closed
- Supported categories are: `Added`, `Fixed`, `Changed`, `Removed`, `Security`, `Deprecated`

### How entries are processed

This action generates **two different outputs** with different processing rules:

| Feature                          | GitHub Release              | Website Changelog      |
| -------------------------------- | --------------------------- | ---------------------- |
| **Audience**                     | Internal (developers, team) | Public (end users)     |
| **[PRIVATE] entries**            | ✅ Included (tag removed)   | ❌ Completely excluded |
| **Issue references** `[#123]`    | ✅ Kept                     | ❌ Removed             |
| **Issue links** `[#123](url)`    | ✅ Kept (clickable)         | ❌ Removed             |
| **Sub-items** (indented)         | ✅ Preserved                | ✅ Preserved           |

**Examples:**

```markdown
# Original CHANGELOG.md entry:

- **[PRIVATE]** [#1001](https://github.com/.../issues/1001) Fixed internal bug

# GitHub Release output:

- [#1001](https://github.com/.../issues/1001) Fixed internal bug

# Website Changelog output:

(entry completely excluded)
```

```markdown
# Original CHANGELOG.md entry:

- [#1000](https://github.com/.../issues/1000) **New Feature**: Added dark mode
  - Automatic theme detection
  - Manual theme toggle

# GitHub Release output:

- [#1000](https://github.com/.../issues/1000) **New Feature**: Added dark mode
  - Automatic theme detection
  - Manual theme toggle

# Website Changelog output:

- **New Feature**: Added dark mode
  - Automatic theme detection
  - Manual theme toggle
```

```markdown
# Original CHANGELOG.md entry:

- [#758] API Keys Page Refactor: Migrated page to TypeScript

# GitHub Release output:

- [#758] API Keys Page Refactor: Migrated page to TypeScript

# Website Changelog output:

- API Keys Page Refactor: Migrated page to TypeScript
```

## How it works

### Automatic trigger (tag push)

When you push a tag starting with `v` (e.g., `v1.2.3`):

```bash
git tag v1.2.3
git push origin v1.2.3
```

The action will:

1. Create a GitHub Release with the full changelog for the version
2. Generate a JSON file with all user-facing entries (excluding those marked as `**[PRIVATE]**`)
3. Create a PR on the website repository
4. Close all issues referenced in the changelog

### Output JSON

The workflow will generate a JSON file structured as follows:

```json
{
  "system": "Dashboard",
  "date": "2026-01-01",
  "version": "1.2.3",
  "features": [
    {
      "type": "text",
      "content": "- **Advanced Query Builder**: Introduced a visual query builder interface.\n  - Added drag-and-drop table joining interface.\n  - Implemented smart autocomplete.",
      "category": "Added"
    }
  ]
}
```

### Manual trigger (for testing)

You can test the action manually from the "Actions" tab on GitHub without creating a real tag. Just enter a test version (e.g., `99.9.9`).

## Requirements

- The repository must have a `CHANGELOG.md` file in the root
- A Personal Access Token with write permissions to the target website repository is required
- The changelog format must follow the structure specified above

## Security

This is a **reusable workflow** that runs with a caller-supplied, **write-capable** token (`personal_access_token`). Treat it like any third-party code you grant credentials to.

### The token value is not leaked — but the workflow code has authority over it

The workflow never prints, echoes, or stores the token; GitHub also masks secret values in logs. **However**, the token is handed to this workflow's code at runtime, and that code uses it to check out the website repo, create/merge changelog PRs, and open a promotion PR. So the risk is not "the secret string escapes" — it's "whatever code runs under this workflow can act with your token." That makes **which version of the workflow you run** a security decision.

### Pin to a reviewed version — do not use `@main`

```yaml
# ❌ Risky: always runs whatever is currently on main
uses: sqlitecloud/changelog-action/.github/workflows/action.yml@main

# ✅ Better: a reviewed, immutable-ish release tag
uses: sqlitecloud/changelog-action/.github/workflows/action.yml@v1

# ✅ Strongest: a full commit SHA (cannot be moved)
uses: sqlitecloud/changelog-action/.github/workflows/action.yml@1a2b3c4d5e6f7890abcdef1234567890abcdef12
```

If you reference `@main`, a compromised or even accidentally-pushed change to this repo's `main` will run **new, unreviewed code with your write token** on your next release. A tag can be re-pointed by a maintainer, so a full commit SHA is the strongest guarantee; a tag plus the branch protections below is a reasonable middle ground.

### Protect this action repository

Maintainers of this repo should:

- Protect `main` with **required pull request reviews** and required status checks.
- Add a **CODEOWNERS** entry for `.github/workflows/action.yml` so changes to the workflow require review by a designated owner.
- Restrict who can push tags / create releases, since consumers pin to tags.

### Least privilege for the token

- Prefer a **fine-grained PAT** or a **GitHub App installation token** scoped to **only the target website repository**, with the minimum permissions needed (contents + pull requests). Avoid classic PATs with broad `repo`/org-wide scope.
- Rotate the token periodically and store it as an org/repo secret — never in a workflow file.
- On the **target website repo**, protect the base branch (e.g. `main`) and the staging branch so the auto-merge/promotion flow can't bypass review where you don't want it to. The staging and base branches are configurable via the `stage_branch` / `base_branch` inputs.
