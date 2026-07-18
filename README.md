# gha-obsidian

Reusable [composite GitHub Actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
for [Obsidian](https://obsidian.md) plugins. One place to maintain CI, e2e,
and release automation; every plugin repo just references these actions from
its own workflow steps and picks up improvements automatically.

The release flow is designed around the [Obsidian community plugin
requirements](https://docs.obsidian.md/Plugins/Releasing/Release+your+plugin+with+GitHub+Actions):
a GitHub release tagged with the exact version from `manifest.json` (no `v`
prefix), with `main.js`, `manifest.json`, and `styles.css` (if present)
attached as individual assets.

## Usage

Each plugin repo keeps its own thin workflow files (composite actions can't
set `permissions:` or `runs-on:`, so those stay in the caller's workflow).

`.github/workflows/ci.yml` — lint, build, test, and (optionally) e2e-test on
every push and PR:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: laughedelic/gha-obsidian/actions/ci@main

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: laughedelic/gha-obsidian/actions/e2e@main

  e2e-mobile:
    runs-on: ubuntu-latest
    steps:
      - uses: laughedelic/gha-obsidian/actions/e2e@main
        with:
          mobile: true
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
    runs-on: ubuntu-latest
    steps:
      - uses: laughedelic/gha-obsidian/actions/release@main
```

Releasing a new version is then just: bump `version` in `manifest.json`
(and `package.json` / `versions.json` as appropriate), merge to `main`, done.
The release action is idempotent — if a release for that version already
exists, it skips instead of failing.

## What the actions do

### `actions/ci`

- `npm ci`
- `npm run lint` (skipped if the script doesn't exist)
- `npm run build`
- `npm test` (skipped if the script doesn't exist)

| Input          | Default | Description                                                                          |
| -------------- | ------- | ------------------------------------------------------------------------------------ |
| `node-version` | _(none)_ | Node.js version to use. Left empty by default, so [`setup-node`](https://github.com/actions/setup-node) falls back to whatever Node is pre-installed on the runner. |

### `actions/e2e`

Runs [wdio-obsidian-service](https://github.com/jesse-r-s-hines/wdio-obsidian-service)
e2e tests. Obsidian is Electron-based and needs a display even in "headless"
mode, so tests run under `xvfb-run`. Caches `.obsidian-cache` (downloaded
Obsidian binaries) across runs.

| Input                 | Default            | Description                                                                          |
| ---------------------- | ------------------ | ------------------------------------------------------------------------------------ |
| `node-version`         | _(none)_            | Node.js version to use. Left empty by default, so `setup-node` falls back to whatever Node is pre-installed on the runner. |
| `mobile`               | `false`             | Run the mobile e2e script instead of desktop                                         |
| `test-script`          | `"test:e2e"`        | npm script to run for desktop e2e tests                                              |
| `test-script-mobile`   | `"test:e2e:mobile"` | npm script to run for mobile e2e tests                                               |

### `actions/release`

- `npm ci`, `npm run build`, `npm test` (if present)
- Generates [build provenance attestations](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations)
  for the release assets
- Creates a GitHub release named after `manifest.json`'s `version`, with
  auto-generated notes and the plugin assets attached

| Input          | Default | Description                                                                          |
| -------------- | ------- | ------------------------------------------------------------------------------------ |
| `node-version` | _(none)_ | Node.js version to use. Left empty by default, so `setup-node` falls back to whatever Node is pre-installed on the runner. |

To pin a specific version, pass it explicitly, e.g. `node-version: "22"` or
`node-version: "lts/*"`.

## Assumptions

- npm with a committed `package-lock.json` (`npm ci`)
- `npm run build` produces `main.js` at the repo root
- `manifest.json` at the repo root is the source of truth for the version
- `actions/e2e` assumes `wdio-obsidian-service`-style `test:e2e` /
  `test:e2e:mobile` npm scripts (override via inputs if named differently)

## Versioning

Plugin repos reference `@main` to always get the latest action. If you
prefer stability over automatic updates, pin to a tag or commit SHA instead:

```yaml
uses: laughedelic/gha-obsidian/actions/release@v1
```

## Why composite actions instead of reusable workflows?

The previous `workflow_call` reusable workflows caused release assets to
fail attestation verification. When a `workflow_call` job runs
`actions/attest-build-provenance`, the OIDC token's signer identity
(`job_workflow_ref`, surfaced as the certificate's SAN / `buildSignerURI`)
is bound to *this* repo's workflow file
(`laughedelic/gha-obsidian/.github/workflows/release.yml`), not to the
calling plugin repo, even though `sourceRepositoryURI` correctly shows the
plugin repo. Verifying against the plugin repo without also passing
`--signer-repo laughedelic/gha-obsidian` fails:

```
$ gh attestation verify main.js --repo laughedelic/some-plugin
Error: verifying with issuer "sigstore.dev"

$ gh attestation verify main.js --repo laughedelic/some-plugin \
    --signer-repo laughedelic/gha-obsidian
✓ verified
```

This is documented, expected behavior for reusable workflows (see GitHub's
["Using artifact attestations and reusable workflows to achieve SLSA v1
Build Level 3"](https://github.blog/security/supply-chain-security/using-artifact-attestations-to-establish-provenance-for-builds/)) —
the signer is supposed to be the trusted centralized builder. But automated
checks that naively compare signer repo to source repo (without knowing
about `--signer-repo`) reject the result.

Composite actions don't have this problem: their steps execute *inside the
caller's own job*, so `job_workflow_ref` stays whatever workflow file the
caller repo defines. Attesting from `actions/release` run as a step in a
plugin repo's own `.github/workflows/release.yml` produces a signer identity
that matches the plugin repo directly.
