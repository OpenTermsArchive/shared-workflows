# Shared Workflows

Reusable GitHub Actions workflows for Open Terms Archive npm packages.

## Available Workflows

### `release-decision.yml`

Decides whether the release process should proceed based on the commit context.

#### Outputs
- `should-release` — `true` if the commit is not from the release bot and comes from a merged PR

---

### `changelog.yml`

Validates the changelog format.

#### Outputs
- `release-type` — `major`, `minor`, `patch`, or `no-release`

---

### `commit.yml`

Lints commit messages using commitlint.

#### Prerequisites
- `@commitlint/cli` in devDependencies
- `commit-messages:lint` script in package.json

---

### `release.yml`

Performs the actual release: updates changelog, bumps version, creates tag, publishes to npm, creates GitHub release.

#### Inputs
- `npm` — string, default `'publish'`
  - `'publish'` — update package.json + publish to npm
  - `'version'` — update package.json only (for private packages)
  - `'none'` — no package.json (GitHub release only)

#### Required permissions
When using `npm: publish`, the calling workflow must declare `id-token: write` for npm provenance.

#### Outputs
- `released` — `true` if a release was made
- `version` — the version that was released

#### Secrets
- `RELEASE_BOT_GITHUB_TOKEN`

---

### `clean-changelog.yml`

Cleans up the changelog when no release is needed.

#### Secrets
- `RELEASE_BOT_GITHUB_TOKEN`

## Dependency Graph

The shared workflows are independent building blocks. Each project assembles them by defining dependencies (`needs:`) and conditions (`if:`) in its own `release.yml`.

Recommended orchestration:

```
release-decision
      │
      ▼
  changelog
      │
      ├──────────────────┐
      ▼                  ▼
     test          clean-changelog
   (local)         (if no-release)
      │
      ▼
   release
      │
      ▼
 [custom steps]
 (e.g. trigger-docs)
```

## Usage

Each project assembles the shared workflows with its own test workflow:

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches:
      - main

concurrency:
  group: release
  cancel-in-progress: false

permissions:
  id-token: write
  contents: read

jobs:
  release-decision:
    uses: OpenTermsArchive/shared-workflows/.github/workflows/release-decision.yml@main

  changelog:
    needs: release-decision
    if: ${{ needs.release-decision.outputs.should-release == 'true' }}
    uses: OpenTermsArchive/shared-workflows/.github/workflows/changelog.yml@main

  test:
    needs: changelog
    if: ${{ needs.changelog.outputs.release-type != 'no-release' }}
    uses: ./.github/workflows/test.yml  # Your project's test workflow

  release:
    needs: [changelog, test]
    if: ${{ needs.changelog.outputs.release-type != 'no-release' }}
    uses: OpenTermsArchive/shared-workflows/.github/workflows/release.yml@main
    secrets: inherit

  clean-changelog:
    needs: changelog
    if: ${{ needs.changelog.outputs.release-type == 'no-release' }}
    uses: OpenTermsArchive/shared-workflows/.github/workflows/clean-changelog.yml@main
    secrets: inherit
```

### For private npm packages (no publish)

```yaml
  release:
    needs: [changelog, test]
    if: ${{ needs.changelog.outputs.release-type != 'no-release' }}
    uses: OpenTermsArchive/shared-workflows/.github/workflows/release.yml@main
    with:
      npm: version
    secrets: inherit
```

### For repos without package.json (GitHub release only)

```yaml
  release:
    needs: changelog
    if: ${{ needs.changelog.outputs.release-type != 'no-release' }}
    uses: OpenTermsArchive/shared-workflows/.github/workflows/release.yml@main
    with:
      npm: none
    secrets: inherit
```

### With custom steps after release (e.g., trigger docs deploy)

```yaml
  release:
    # ... same as above

  trigger-docs:
    needs: release
    if: ${{ needs.release.outputs.released == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - name: Trigger documentation deploy
        uses: peter-evans/repository-dispatch@v2
        with:
          token: ${{ secrets.TRIGGER_DOCS_DEPLOY_TOKEN }}
          event-type: engine-release
          repository: OpenTermsArchive/docs
          client-payload: '{"version": "v${{ needs.release.outputs.version }}"}'
```

### Changelog validation on PRs

```yaml
# .github/workflows/changelog.yml
name: Changelog

on:
  pull_request:

jobs:
  validate:
    uses: OpenTermsArchive/shared-workflows/.github/workflows/changelog.yml@main
```

### Commit message linting on PRs

```yaml
# .github/workflows/commit.yml
name: Lint commit messages

on:
  pull_request:

jobs:
  commitlint:
    uses: OpenTermsArchive/shared-workflows/.github/workflows/commit.yml@main
```
