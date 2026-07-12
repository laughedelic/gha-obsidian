# gha-obsidian

Reusable GitHub Actions workflows for [Obsidian](https://obsidian.md) plugins.
One place to maintain CI and release automation; every plugin repo just calls
these workflows and picks up improvements automatically.

The release flow is designed around the [Obsidian community plugin
requirements](https://docs.obsidian.md/Plugins/Releasing/Release+your+plugin+with+GitHub+Actions):
a GitHub release tagged with the exact version from `manifest.json` (no `v`
prefix), with `main.js`, `manifest.json`, and `styles.css` (if present)
attached as individual assets.

## Usage

Add these two files to your plugin repo.

`.github/workflows/ci.yml` — lint, build, and test on every push and PR:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    uses: laughedelic/gha-obsidian/.github/workflows/ci.yml@main
```

`.github/workflows/release.yml` — cut a release whenever the version in
`manifest.json` changes on `main`:

```yaml
name: Release

on:
  push:
    branches: [main]
    paths: [manifest.json]

permissions:
  contents: write     # create the release
  id-token: write     # build provenance attestation
  attestations: write # build provenance attestation

jobs:
  release:
    uses: laughedelic/gha-obsidian/.github/workflows/release.yml@main
```

Releasing a new version is then just: bump `version` in `manifest.json`
(and `package.json` / `versions.json` as appropriate), merge to `main`, done.
The workflow is idempotent — if a release for that version already exists,
it skips instead of failing.

## What the workflows do

### `ci.yml`

- `npm ci`
- `npm run lint` (skipped if the script doesn't exist)
- `npm run build`
- `npm test` (skipped if the script doesn't exist)

### `release.yml`

- `npm ci`, `npm run build`, `npm test` (if present)
- Generates [build provenance attestations](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations)
  for the release assets
- Creates a GitHub release named after `manifest.json`'s `version`, with
  auto-generated notes and the plugin assets attached

## Inputs

Both workflows accept:

| Input          | Default | Description            |
| -------------- | ------- | ---------------------- |
| `node-version` | `"20"`  | Node.js version to use |

```yaml
jobs:
  check:
    uses: laughedelic/gha-obsidian/.github/workflows/ci.yml@main
    with:
      node-version: "22"
```

## Assumptions

- npm with a committed `package-lock.json` (`npm ci`)
- `npm run build` produces `main.js` at the repo root
- `manifest.json` at the repo root is the source of truth for the version

## Versioning

Plugin repos reference `@main` to always get the latest workflow. If you
prefer stability over automatic updates, pin to a tag or commit SHA instead:

```yaml
uses: laughedelic/gha-obsidian/.github/workflows/release.yml@v1
```
