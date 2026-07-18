# gha-obsidian

Reusable GitHub Actions for [Obsidian](https://obsidian.md) plugin CI and
release automation. One place to maintain lint/build/test/e2e/release logic;
plugin repos pick up improvements automatically instead of copy-pasting
workflow YAML around.

The release flow is designed around the [Obsidian community plugin
requirements](https://docs.obsidian.md/Plugins/Releasing/Release+your+plugin+with+GitHub+Actions):
a GitHub release tagged with the exact version from `manifest.json` (no `v`
prefix), with `main.js`, `manifest.json`, and `styles.css` (if present)
attached as individual assets.

There are three independent action/workflow pairs, one per concern:

| Concern    | Composite action        | Reusable workflow                 |
| ---------- | ------------------------ | ---------------------------------- |
| CI checks  | `actions/ci-checks`      | `.github/workflows/ci-checks.yml`  |
| E2E tests  | `actions/e2e-test`       | `.github/workflows/e2e-test.yml`   |
| Release    | `actions/release`        | `.github/workflows/release.yml`    |

- **Reusable workflows** are the typical, job-level way to use this. They
  check out the repo, set up Node.js, and run the corresponding composite
  action. Use these unless you need something the typical job doesn't do.
- **Composite actions** are the underlying building blocks, invoked at the
  step level. They're intentionally minimal about environment setup — no
  checkout, no `setup-node` — since that's common boilerplate any caller
  already has. But each action still owns everything specific to its own
  concern (e.g. `actions/e2e-test` handles its own build, its own Obsidian
  binary cache, its own optional window manager) rather than pushing that
  out to the caller. Use these directly when you need to interleave extra
  steps of your own around them.

## Usage: reusable workflows (typical)

`.github/workflows/ci.yml` — lint, build, test, and e2e-test on every push
and PR:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    uses: laughedelic/gha-obsidian/.github/workflows/ci-checks.yml@main
    with:
      node-version: "22"

  e2e:
    uses: laughedelic/gha-obsidian/.github/workflows/e2e-test.yml@main
    with:
      node-version: "22"
      test-script: "test:e2e"
      versions-lock-command: "npx tsx wdio.conf.mts 2>/dev/null | grep 'obsidian-cache-key:'"
      window-manager: true # if your tests need real focus/hover behavior

  e2e-mobile:
    uses: laughedelic/gha-obsidian/.github/workflows/e2e-test.yml@main
    with:
      node-version: "22"
      test-script: "test:e2e:mobile"
```

`.github/workflows/release.yml` — cut a release whenever the version in
`manifest.json` changes on `main`:

```yaml
name: Release

on:
  push:
    branches: [main]
    paths: [manifest.json]

jobs:
  release:
    uses: laughedelic/gha-obsidian/.github/workflows/release.yml@main
    with:
      node-version: "22"
```

Releasing a new version is then just: bump `version` in `manifest.json`
(and `package.json` / `versions.json` as appropriate), merge to `main`, done.
The release workflow is idempotent — if a release for that version already
exists, it skips instead of failing.

All three reusable workflows accept a `node-version` input. Leave it unset
to let [`setup-node`](https://github.com/actions/setup-node) fall back to
whatever Node is pre-installed on the runner, or pass it explicitly to pin a
version, e.g. `node-version: "22"` or `node-version: "lts/*"`.

`e2e-test.yml` runs one e2e npm script per call — invoke it once per script
if you have separate desktop/mobile suites, as in the example above.

## Usage: composite actions (custom jobs)

Use these directly, at the step level, when your job needs more than the
typical checkout → setup-node → run pipeline — e.g. extra tooling, or a
different step order:

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v6
        with: { node-version: "22", cache: npm }

      - run: sudo apt-get install -y some-extra-tool

      - uses: laughedelic/gha-obsidian/actions/e2e-test@main
        with:
          test-script: test:e2e
          versions-lock-command: "npx tsx wdio.conf.mts 2>/dev/null | grep 'obsidian-cache-key:'"
          window-manager: true # if your tests need real focus/hover behavior
```

## What's included

### `actions/ci-checks`

- `npm ci`
- `npm run lint` (skipped if the script doesn't exist)
- `npm run build`
- `npm test` (skipped if the script doesn't exist)

No inputs.

### `actions/e2e-test`

- `npm ci`
- Optionally runs `versions-lock-command` and hashes its output for the
  Obsidian binary cache key
- `npm run build`
- Caches the downloaded Obsidian binary (`.obsidian-cache`)
- Optionally installs and starts `herbstluftwm` + `dzen2` under Xvfb, for
  tests that need real window management (e.g. focus/hover behavior) rather
  than a bare X server
- Runs the e2e npm script under `xvfb-run` (Obsidian is Electron and needs a
  display even when "headless")

| Input                    | Default      | Description                                                                                                                                             |
| ------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `test-script`             | _(required)_   | npm script to run for the e2e tests. Call the action/workflow once per script for separate desktop/mobile suites.                                       |
| `versions-lock-command`   | _(none)_       | Shell command run after `npm ci`, before build/cache; its stdout is hashed for the Obsidian binary cache key. Leave empty for a static, non-version-aware entry. |
| `window-manager`          | `false`        | Install and run `herbstluftwm` + `dzen2` under Xvfb                                                                                                      |

### `actions/release`

- `npm ci`, `npm run build`, `npm test` (if present)
- Generates [build provenance attestations](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations)
  for the release assets
- Creates a GitHub release named after `manifest.json`'s `version`, with
  auto-generated notes and the plugin assets attached

No inputs.

## Assumptions

- npm with a committed `package-lock.json` (`npm ci`)
- `npm run build` produces `main.js` at the repo root
- `manifest.json` at the repo root is the source of truth for the version
- `actions/e2e-test` runs whatever npm script `test-script` names — no
  assumption about naming (the `e2e-test.yml` workflow defaults it to
  `test:e2e`)

## Versioning

Plugin repos reference `@main` to always get the latest version. If you
prefer stability over automatic updates, pin to a tag or commit SHA instead:

```yaml
uses: laughedelic/gha-obsidian/.github/workflows/release.yml@v1
```
