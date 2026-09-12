# ci/

The canonical GitHub Actions workflows now live under `.github/workflows/` and
are executed automatically:

- `.github/workflows/ci.yml` — fmt, build, Rust tests, clippy, jsdom, docs lint
- `.github/workflows/build-windows.yml` — Windows release build artifact + SHA-256
- `.github/workflows/e2e.yml` — scheduled/manual WinDivert field smoke test
- `.github/workflows/release.yml` — semver-tagged Windows release assets

The YAML files in this directory are retained as templates for environments
that import workflow definitions from `ci/`. They are **not** the files GitHub
runs in this checkout, so do not cite them as evidence that CI executed.

The Windows artifact never bundles WinDivert.dll or WinDivert64.sys. Fetch the
official pinned driver with `scripts/fetch-windivert.ps1` and verify the hashes
before running the executable.
