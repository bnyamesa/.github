# Finachub Shared Workflows

Centralised CI/CD pipeline for all Finachub WordPress plugins.
One pipeline definition — all plugins inherit it automatically.

## Plugins using this pipeline

| Plugin | Repo |
|--------|------|
| Finachub License Manager | bnyamesa/finachub-license-manager |
| Finachub Mobile Money | bnyamesa/finachub-mobile-money |
| Dashboard and Analytics for WooCommerce Pro | bnyamesa/dashboard-and-Analytics-for-WooCommerce |

## Pipeline stages

```
lint ──► test (PHP 7.4 / 8.1 / 8.2) ──► build ──► deploy  (tags only)
```

| Stage | What it does |
|-------|-------------|
| **lint** | PHPCS coding standards · PHPStan static analysis · Composer CVE audit · License compatibility check |
| **test** | PHPUnit across PHP 7.4, 8.1, and 8.2 in parallel (`fail-fast: false`) |
| **build** | Assembles production ZIP · verifies integrity · generates versioned update JSON and release notes |
| **deploy** | Atomic upload to update server · rotates old ZIPs · smoke-tests the live URL · creates GitHub Release |

## How to add a new plugin

Create `.github/workflows/deploy.yml` in the plugin repo:

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
    uses: bnyamesa/.github/.github/workflows/plugin-pipeline.yml@main
    with:
      slug:               your-plugin-slug
      main_file:          your-plugin-file.php
      plugin_name:        Your Plugin Name
      plugin_description: One-line description for the update JSON.
      phpstan_level:      0
    secrets: inherit
```

Then add these three secrets in the plugin repo under **Settings → Secrets and variables → Actions**:

| Secret | Value |
|--------|-------|
| `HOST` | Update server hostname |
| `USERNAME` | SSH username on the update server |
| `SSH_PRIVATE_KEY` | SSH private key (PEM format) |

## Available inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `slug` | yes | — | Plugin directory slug |
| `main_file` | yes | — | Main plugin PHP filename |
| `plugin_name` | yes | — | Human-readable name (used in update JSON) |
| `plugin_description` | yes | — | Short description (used in update JSON) |
| `phpstan_level` | no | `0` | PHPStan strictness level (0–9) |
| `phpstan_extra_args` | no | `""` | Extra phpstan flags (e.g. `--memory-limit=-1`) |
| `update_server_base_url` | no | `https://finachub.com/updates` | Public base URL where files are served |
| `update_server_path` | no | `/srv/www/wordpress/updates` | Absolute path on the update server |
| `keep_releases` | no | `3` | How many old versioned ZIPs to keep on the server |

## Deployment details

- **Trigger** — push a tag matching `v*` (e.g. `v2.5.1`)
- **Version guard** — the tag must match `Version:` in the plugin header; mismatches block the build
- **Atomic swap** — files land in `.incoming/` then `mv` into the live directory; customers never see a partial state
- **Release rotation** — keeps the `keep_releases` most recent versioned ZIPs, deletes the rest automatically
- **Smoke test** — curls the live update JSON after deploy and asserts the version string matches
- **GitHub Release** — created automatically with release notes from `git log` since the previous tag

## Updating the pipeline

Edit `.github/workflows/plugin-pipeline.yml` in this repo.
All plugins pick up the change on their next push — no edits needed in individual plugin repos.
