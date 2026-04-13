# Finachub Shared Plugin Pipeline

One CI/CD pipeline definition that every Finachub WordPress plugin inherits automatically.
Changes made here flow to all plugins on their next push — no edits needed in individual repos.

---

## Table of contents

1. [How it works](#how-it-works)
2. [Plugins using this pipeline](#plugins-using-this-pipeline)
3. [Pipeline stages](#pipeline-stages)
4. [Adding a new plugin](#adding-a-new-plugin)
5. [Required secrets](#required-secrets)
6. [All available inputs](#all-available-inputs)
7. [Releasing a new version](#releasing-a-new-version)
8. [Deployment details](#deployment-details)
9. [Upgrading to a new pipeline version](#upgrading-to-a-new-pipeline-version)
10. [Updating the pipeline itself](#updating-the-pipeline-itself)
11. [Troubleshooting](#troubleshooting)

---

## How it works

Each plugin repo contains a thin `deploy.yml` file (about 20 lines) that simply calls the reusable pipeline defined here:

```
Plugin repo                          This repo (bnyamesa/.github)
─────────────────────────────        ─────────────────────────────────────────
.github/workflows/deploy.yml   ───►  .github/workflows/plugin-pipeline.yml@v1
  (calls the shared pipeline)          (the actual pipeline logic lives here)
```

The plugin's `deploy.yml` passes in a few values specific to that plugin (slug, main file, name, etc.).
Everything else — linting, testing, building, deploying — is handled by the shared pipeline.

---

## Plugins using this pipeline

| Plugin | Repository | Pipeline version |
|--------|-----------|-----------------|
| Finachub License Manager | [bnyamesa/finachub-license-manager](https://github.com/bnyamesa/finachub-license-manager) | `@v1` |
| Finachub Mobile Money | [bnyamesa/finachub-mobile-money](https://github.com/bnyamesa/finachub-mobile-money) | `@v1` |
| Dashboard and Analytics for WooCommerce Pro | [bnyamesa/dashboard-and-Analytics-for-WooCommerce](https://github.com/bnyamesa/dashboard-and-Analytics-for-WooCommerce) | `@v1` |

---

## Pipeline stages

```
lint ──► test (PHP 7.4 / 8.1 / 8.2) ──► build ──► deploy  (tags only)
```

Each stage must pass before the next one begins. The `test` stage runs across three PHP versions in parallel.

| Stage | Runs on | What it does |
|-------|---------|-------------|
| **lint** | PHP 8.2 | PHPCS coding standards check, PHPStan static analysis, Composer CVE audit, license compatibility scan |
| **test** | PHP 7.4, 8.1, 8.2 | PHPUnit test suite — all three versions run in parallel, none blocked by another failing |
| **build** | PHP 7.4 | Assembles the production ZIP, checks ZIP integrity, generates the update JSON and release notes, uploads everything as a GitHub Actions artifact |
| **deploy** | — | Validates secrets, uploads files to the update server atomically, runs a smoke test (with auto-retry), rolls back if the smoke test fails, creates a GitHub Release |

> **Deploy only runs on version tags.** Pushes to `main` and pull requests run lint, test, and build only — they never touch the live server.

---

## Adding a new plugin

### Step 1 — Create the caller workflow

Create `.github/workflows/deploy.yml` inside the plugin repo:

```yaml
name: Finachub CI/CD

on:
  push:
    branches: [ main ]
    tags:    [ 'v*' ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  pipeline:
    uses: bnyamesa/.github/.github/workflows/plugin-pipeline.yml@v1
    with:
      slug:               your-plugin-slug
      main_file:          your-plugin-file.php
      plugin_name:        Your Plugin Name
      plugin_description: One-line description for the update JSON.
      phpstan_level:      0
    secrets: inherit
```

Replace the four `with:` values with the plugin's actual details.
For PHPStan level, start at `0` (errors only) and raise it over time as code quality improves.

### Step 2 — Add the required secrets

In the **plugin repo** (not this repo), go to **Settings → Secrets and variables → Actions** and add:

| Secret | What to put |
|--------|------------|
| `HOST` | Hostname of the update server (e.g. `finachub.com`) |
| `USERNAME` | SSH username used to connect to the update server |
| `SSH_PRIVATE_KEY` | The private SSH key in PEM format (starts with `-----BEGIN`) |

### Step 3 — Commit a `composer.lock`

The pipeline caches Composer packages using the lockfile as a cache key.
Make sure `composer.lock` is committed and **not** listed in `.gitignore`.

If it is in `.gitignore`, remove that line, then run:

```bash
composer install   # generates / confirms the lockfile
git add composer.lock
git commit -m "chore: commit composer.lock for reproducible builds"
```

### Step 4 — Set up a `production` environment (optional but recommended)

The deploy job targets a GitHub Environment named `production`.
If you configure it in **Settings → Environments**, you can add required reviewers —
meaning a human must approve before any deploy to the live server goes ahead.

If no `production` environment is configured, deploys proceed automatically.

---

## Required secrets

Set these in each **plugin repo** under Settings → Secrets and variables → Actions.

| Secret | Description |
|--------|-------------|
| `HOST` | Update server hostname |
| `USERNAME` | SSH username on the update server |
| `SSH_PRIVATE_KEY` | SSH private key (PEM format) for connecting to the update server |

> `GITHUB_TOKEN` is provided automatically by GitHub — you do not need to set it.

---

## All available inputs

Pass these under the `with:` block in the plugin's `deploy.yml`.

| Input | Required | Default | Description |
|-------|:--------:|---------|-------------|
| `slug` | yes | — | Plugin directory slug, e.g. `finachub-mobile-money`. Used as the ZIP prefix and in file paths on the server. |
| `main_file` | yes | — | Main plugin PHP filename, e.g. `finachub-mobile-money.php`. The pipeline reads the `Version:` header from this file. |
| `plugin_name` | yes | — | Human-readable plugin name. Appears in the update JSON served to WordPress sites. |
| `plugin_description` | yes | — | Short one-line description. Appears in the update JSON. |
| `phpstan_level` | no | `0` | PHPStan strictness level, 0 (most permissive) to 9 (strictest). Start at 0 and raise gradually. |
| `phpstan_extra_args` | no | `""` | Extra flags passed to PHPStan, e.g. `--memory-limit=-1` for large codebases. |
| `update_server_base_url` | no | `https://finachub.com/updates` | Public URL where the update server serves ZIPs and JSON files. Used to build `download_url` in the update JSON. |
| `update_server_path` | no | `/srv/www/wordpress/updates` | Absolute filesystem path on the update server where files are deployed. |
| `keep_releases` | no | `3` | How many previous versioned ZIPs to keep on the server. Older ones are deleted automatically after each deploy. |

---

## Releasing a new version

1. **Bump the version** in the plugin's main PHP file header:
   ```php
   * Version: 2.6.0
   ```
   Make sure the `Version:` header and any version constant in the file match.

2. **Commit** all changes to `main`.

3. **Push a tag** that matches the version:
   ```bash
   git tag v2.6.0
   git push origin v2.6.0
   ```

4. The pipeline automatically:
   - Verifies the tag matches the `Version:` header (fails if they differ)
   - Builds a ZIP named `{slug}-2.6.0.zip`
   - Generates `{slug}-updates.json` with the new version
   - Deploys atomically to the update server
   - Smoke-tests the live URL to confirm the new version is visible
   - Rolls back the update JSON if the smoke test fails
   - Creates a GitHub Release with auto-generated release notes

> **Tip:** Use [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `chore:` prefixes) so the auto-generated release notes are meaningful.

---

## Deployment details

### Where files land on the server

All plugins deploy to the same flat directory (`update_server_path`). Each plugin's files are prefixed with its slug so they never collide:

```
/srv/www/wordpress/updates/
  finachub-license-manager-2.6.0.zip
  finachub-license-manager-updates.json
  finachub-mobile-money-1.3.0.zip
  finachub-mobile-money-updates.json
  ...
```

WordPress sites fetch `{slug}-updates.json` to check for updates, then download the ZIP.

### Atomic deployment

Files are never written directly into the live directory. The flow is:

```
1. SCP files to  /srv/.../updates/.incoming/
2. SSH mv .incoming/{zip}          → updates/{zip}       (atomic)
3. SSH mv .incoming/{slug}-updates.json → updates/{slug}-updates.json  (atomic)
```

Customers fetching the update JSON during a deploy always get either the old version or the new version — never a partial state.

### Smoke test with retry

After deploy, the pipeline fetches the live update JSON and checks that the `version` field matches what was just deployed. It retries up to **10 times** with a **5-second wait** between attempts, to allow for any server-side propagation delay. If all 10 attempts fail, the deploy job fails.

### Automatic rollback

Before the atomic swap, the pipeline saves a backup of the current update JSON. If the smoke test fails, the backup is restored automatically, so the live update JSON always reflects a version that is actually available for download.

### Release rotation

After each deploy, old versioned ZIPs for that slug are pruned. Only the `keep_releases` most recent ZIPs are kept (default: 3). The update JSON and the latest ZIP are always kept regardless of this setting.

### Version guard

The pipeline reads the `Version:` header from the plugin's main PHP file and compares it to the git tag. If they do not match exactly, the build fails before anything is uploaded. This prevents deploying a ZIP labelled `v2.6.0` that actually contains version `2.5.9` code.

---

## Upgrading to a new pipeline version

When a new pipeline version is released (e.g. `v2`), update the `uses:` line in the plugin's `deploy.yml`:

```yaml
# Before
uses: bnyamesa/.github/.github/workflows/plugin-pipeline.yml@v1

# After
uses: bnyamesa/.github/.github/workflows/plugin-pipeline.yml@v2
```

Plugins pinned to `@v1` will continue using the v1 pipeline unchanged until you opt them in.
Read the release notes for the new version before upgrading to check for any breaking changes.

---

## Updating the pipeline itself

1. Edit `.github/workflows/plugin-pipeline.yml` in **this repo**.
2. Test the change by pushing to `main` — any plugin pinned to `@main` will pick it up immediately.
3. Once satisfied, create a new tag:
   ```bash
   git tag v2
   git push origin v2
   ```
4. Update each plugin's `deploy.yml` from `@v1` to `@v2` when ready.

> **Action version pinning:** All third-party actions in the pipeline are pinned to immutable commit SHAs (not floating version tags like `@v4`). This means the pipeline cannot be silently changed by a third party. When upgrading an action (e.g. after a security advisory), look up the new commit SHA and update the comment alongside it.

---

## Troubleshooting

### Build fails: "Tag version does not match plugin header version"

The git tag you pushed (e.g. `v2.6.0`) does not match the `Version:` line in the plugin's main PHP file. Update the version in the file, commit, delete the tag, and re-push:

```bash
git tag -d v2.6.0
git push origin :refs/tags/v2.6.0
# fix the version in the PHP file, then commit
git tag v2.6.0
git push origin v2.6.0
```

### Deploy fails: "Missing required secrets"

One or more of `HOST`, `USERNAME`, or `SSH_PRIVATE_KEY` are not set in the plugin repo's secrets. Go to **Settings → Secrets and variables → Actions** in that specific plugin repo and add the missing ones.

### Smoke test fails after deploy

The pipeline retried 10 times over ~50 seconds and the live update JSON still did not reflect the new version. The pipeline will have automatically rolled back the update JSON to the previous version. Investigate:
- Is the update server reachable from GitHub Actions?
- Did the `mv` commands in the atomic swap step succeed?
- Check the server logs for permission or disk space issues.

Once fixed, re-run the deploy by pushing a new tag.

### Composer cache miss every run

Make sure `composer.lock` is committed to the repo and not listed in `.gitignore`. The cache key is derived from the lockfile — if it is absent, there is nothing to key on and the cache never hits.

### PHPStan runs out of memory

Add `phpstan_extra_args: --memory-limit=-1` under `with:` in the plugin's `deploy.yml`.
