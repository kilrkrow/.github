# kilrkrow/.github

Shared, reusable GitHub Actions workflows for kilrkrow repositories.

This repository is public on purpose: a public repository can only call
reusable workflows that live in a public repository. It holds **no secret
values**. Every secret (for example `CHOCOLATEY_API_KEY`) lives in the caller
repository's own Actions secrets and is passed explicitly by the caller.

| Workflow | What it does |
| --- | --- |
| [`build-windows-exe.yml`](.github/workflows/build-windows-exe.yml) | Builds a Windows EXE with PyInstaller, zips it, and creates a GitHub Release with the zip attached. |
| [`choco-publish.yml`](.github/workflows/choco-publish.yml) | Packs a Chocolatey package for an existing GitHub release, uploads the `.nupkg` as a run artifact, and pushes it to chocolatey.org when every safety gate passes. |

All actions used inside these workflows are pinned to full commit SHAs, and
caller inputs reach scripts only through `env:`, never interpolated into
PowerShell.

## Versioning policy

- Callers pin a release tag (`@v1`) or a full commit SHA. Never `@main`.
- `v1` is a moving major tag: it gets backward-compatible fixes and new
  optional inputs.
- Breaking changes (a removed or renamed input, a new required input, changed
  defaults or behavior) go to `v2`. `v1` keeps working for existing callers.
- A SHA pin never changes. Use one when you want to review every update
  yourself (Dependabot can bump it).

## build-windows-exe.yml

Checks out the caller at the triggering ref, sets up Python, installs
`extra_pip` plus `requirements.txt`, runs PyInstaller on the spec file, runs the
repo's release script (or zips the dist folder), and creates a GitHub Release
for the tag with that zip attached (`softprops/action-gh-release`, generated
release notes). Trigger it from a `v*` tag push.

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `app_name` | yes | | Executable name (PyInstaller output folder and `.exe` name). |
| `spec_file` | no | `<app_name>.spec` | Path to the `.spec` file. |
| `python_version` | no | `3.12` | Python version. |
| `release_script` | no | `tools/release.ps1` | PS1 that zips the dist folder; receives `-Dist <path>`. If missing, the dist folder is zipped to `release/<app_name>-win64.zip`. |
| `zip_glob` | no | `release/*.zip` | Glob that finds the zip to upload. |
| `extra_pip` | no | `pyinstaller` | Space-separated extra pip packages. |
| `qt_api` | no | (empty) | Sets `QT_API` and `PYQTGRAPH_QT_LIB` during the build. |

Secrets: none. The job needs `contents: write` (to create the release); the
caller must grant it.

```yaml
# .github/workflows/release.yml in the caller repo
name: Build & Release
on:
  push:
    tags: ["v*"]
jobs:
  build:
    uses: kilrkrow/.github/.github/workflows/build-windows-exe.yml@v1
    with:
      app_name: NetPulse
      qt_api: PySide6
    permissions:
      contents: write
```

## choco-publish.yml

1. Checks out the **caller's running commit**, not the release tag. For a
   `workflow_run` or `workflow_dispatch` caller that is the default branch, so
   the nuspec and install script always come from current `main` and an older
   tag can be republished with the current package definition.
2. Downloads the exact release asset `asset_name` from the caller repository
   (`gh release download --repo`) and computes its SHA256. Fails with a clear
   message if the release or asset does not exist (for example when the app
   version baked into the zip name does not match the tag).
3. Replaces `__VERSION__` and `__CHECKSUM__` in the nuspec and the install
   script (literal replacement). In the nuspec, `<version>__VERSION__</version>`
   gets the Chocolatey package version; every other `__VERSION__` gets the
   release version (tag without `v`), which is what download URLs need. For a
   prerelease tag like `v1.2.3-rc.1` the package version is `1.2.3-rc1`
   (Chocolatey labels must be alphanumeric).
4. `choco pack`, then uploads the `.nupkg` as the run artifact
   `<id>-<version>-nupkg` (always, even when the push is skipped), so the
   package can be installed and tested before moderation approves it:
   `choco install <id> --source <folder with the .nupkg> --version <version>`.
5. Pushes to `https://push.chocolatey.org/` unless a gate fails. A failed gate
   logs `Chocolatey push SKIPPED: <reasons>` and the job still succeeds:
   - the tag contains `-` (prerelease/test tag);
   - the repository is private (the package's download URL would not work);
   - `canonical_repo` is set and the run is in another repository;
   - the `CHOCOLATEY_API_KEY` secret is missing.

   A 403 (id owned by someone else, or wrong key) and a 409 (version already
   published) fail the job with an explanation.

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `release_tag` | yes | | Release tag, e.g. `v1.4.3` (`refs/tags/` prefix is stripped). |
| `asset_name` | yes | | Exact asset file name; `{version}` is replaced with the tag without `v`, e.g. `NetPulse-{version}-win64.zip`. No globs. |
| `nuspec_path` | no | the single `choco/*.nuspec` | Nuspec template. The package id is read from its `<id>`. |
| `install_script_path` | no | `tools/chocolateyinstall.ps1` next to the nuspec | Install script template. |
| `canonical_repo` | no | (empty, no check) | `owner/name`; skip the push in any other repository. |
| `artifact_retention_days` | no | `14` | Retention of the `.nupkg` artifact. |

| Secret | Required | Meaning |
| --- | --- | --- |
| `CHOCOLATEY_API_KEY` | no | Chocolatey community API key, stored in the caller repo. Without it the package is packed and uploaded but not pushed. |

Outputs: `package_id`, `package_version`, `nupkg_name`, `pushed` (`true` or
`false`).

The workflow runs with `permissions: contents: read` and a concurrency group
per repository and tag.

```yaml
# .github/workflows/choco.yml in the caller repo
name: Publish to Chocolatey
on:
  workflow_run:
    workflows: ["Build & Release"]
    types: [completed]
  workflow_dispatch:
    inputs:
      tag:
        description: "Existing release tag to publish (e.g. v1.4.2)"
        required: true
        type: string
permissions:
  contents: read
jobs:
  publish:
    if: >-
      github.event_name == 'workflow_dispatch' ||
      (github.event.workflow_run.conclusion == 'success' &&
       startsWith(github.event.workflow_run.head_branch, 'v'))
    uses: kilrkrow/.github/.github/workflows/choco-publish.yml@v1
    with:
      release_tag: ${{ github.event_name == 'workflow_dispatch' && inputs.tag || github.event.workflow_run.head_branch }}
      asset_name: NetPulse-{version}-win64.zip
      canonical_repo: kilrkrow/netpulse
    secrets:
      CHOCOLATEY_API_KEY: ${{ secrets.CHOCOLATEY_API_KEY }}
```

Template placeholders in the caller repo:

```xml
<version>__VERSION__</version>
```

```powershell
url      = 'https://github.com/OWNER/REPO/releases/download/v__VERSION__/App-__VERSION__-win64.zip'
checksum = '__CHECKSUM__'
```
