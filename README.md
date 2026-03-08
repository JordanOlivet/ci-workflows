# ci-workflows

Centralized CI/CD workflows and composite actions for GitHub repositories.

## Overview

This repository provides reusable GitHub Actions workflows and composite actions for:

- **Release management**: Semantic versioning based on PR labels
- **Multi-platform builds**: Rust, .NET, Node.js
- **Docker builds**: Push to GitHub Container Registry
- **GitHub Releases**: Automated release creation with artifacts

## Usage

### Reusable Workflows

Reference workflows using `uses: JordanOlivet/ci-workflows/.github/workflows/<workflow>.yml@v1`

#### release.yml

Handles version bumping, Git tagging, and changelog generation based on PR labels.

```yaml
name: Release
on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  release:
    uses: JordanOlivet/ci-workflows/.github/workflows/release.yml@v1
    with:
      version-source: file  # file, cargo, or npm
      version-file: VERSION
      changelog: true
    secrets: inherit
```

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `version-source` | No | `file` | Version source: `file`, `cargo`, or `npm` |
| `version-file` | No | `VERSION` | Path to version file |
| `changelog` | No | `true` | Generate CHANGELOG.md |
| `commit-version` | No | `true` | Commit version file changes |

**Outputs:**
| Output | Description |
|--------|-------------|
| `version` | New version number |
| `tag` | Git tag (v-prefixed) |
| `should_release` | Whether a release should be created |

#### build-rust.yml

Multi-platform Rust binary builds.

```yaml
jobs:
  build:
    uses: JordanOlivet/ci-workflows/.github/workflows/build-rust.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      platforms: |
        [
          {"os":"windows-latest","target":"x86_64-pc-windows-msvc","artifact":"myapp.exe","asset":"myapp-windows-x86_64.exe"},
          {"os":"ubuntu-latest","target":"x86_64-unknown-linux-gnu","artifact":"myapp","asset":"myapp-linux-x86_64"},
          {"os":"macos-latest","target":"x86_64-apple-darwin","artifact":"myapp","asset":"myapp-macos-x86_64"},
          {"os":"macos-latest","target":"aarch64-apple-darwin","artifact":"myapp","asset":"myapp-macos-arm64"}
        ]
    secrets: inherit
```

#### build-docker.yml

Docker image build and push to GHCR.

```yaml
jobs:
  build-docker:
    uses: JordanOlivet/ci-workflows/.github/workflows/build-docker.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      image-name: my-app
      dockerfile: Dockerfile
      context: .
    secrets: inherit
```

#### build-dotnet.yml

.NET project build and test.

```yaml
jobs:
  build:
    uses: JordanOlivet/ci-workflows/.github/workflows/build-dotnet.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      project-path: src/MyProject
      dotnet-version: '9.0.x'
      run-tests: true
    secrets: inherit
```

#### build-node.yml

Node.js project build and test.

```yaml
jobs:
  build:
    uses: JordanOlivet/ci-workflows/.github/workflows/build-node.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      working-directory: frontend
      node-version: '20'
      package-manager: npm
    secrets: inherit
```

#### gh-release.yml

Create GitHub Release with artifacts.

```yaml
jobs:
  create-release:
    uses: JordanOlivet/ci-workflows/.github/workflows/gh-release.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      tag: ${{ needs.release.outputs.tag }}
      artifacts-path: artifacts
    secrets: inherit
```

### Composite Actions

Use composite actions for more granular control:

```yaml
steps:
  - uses: JordanOlivet/ci-workflows/actions/extract-version@v1
    with:
      source: cargo
      path: Cargo.toml

  - uses: JordanOlivet/ci-workflows/actions/determine-bump@v1
    with:
      labels: ${{ toJson(github.event.pull_request.labels.*.name) }}

  - uses: JordanOlivet/ci-workflows/actions/calculate-version@v1
    with:
      current: ${{ steps.extract.outputs.base_version }}
      bump_type: ${{ steps.bump.outputs.bump_type }}

  - uses: JordanOlivet/ci-workflows/actions/update-version-file@v1
    with:
      source: cargo
      path: Cargo.toml
      version: ${{ steps.calc.outputs.new_version }}
```

## PR Labels

Add these labels to your PRs to trigger releases:

| Label | Effect |
|-------|--------|
| `release-major` | Bump major version (X.0.0) |
| `release-minor` | Bump minor version (x.Y.0) |
| `release-patch` | Bump patch version (x.y.Z) |

## Complete Examples

### Rust CLI Application (claude-cmd)

```yaml
name: Release
on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  release:
    uses: JordanOlivet/ci-workflows/.github/workflows/release.yml@v1
    with:
      version-source: cargo
      version-file: Cargo.toml
      changelog: false
    secrets: inherit

  build:
    needs: release
    if: needs.release.outputs.should_release == 'true'
    uses: JordanOlivet/ci-workflows/.github/workflows/build-rust.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      platforms: |
        [
          {"os":"windows-latest","target":"x86_64-pc-windows-msvc","artifact":"claude-cmd.exe","asset":"claude-cmd-windows-x86_64.exe"},
          {"os":"ubuntu-latest","target":"x86_64-unknown-linux-gnu","artifact":"claude-cmd","asset":"claude-cmd-linux-x86_64"},
          {"os":"macos-latest","target":"x86_64-apple-darwin","artifact":"claude-cmd","asset":"claude-cmd-macos-x86_64"},
          {"os":"macos-latest","target":"aarch64-apple-darwin","artifact":"claude-cmd","asset":"claude-cmd-macos-arm64"}
        ]
    secrets: inherit

  create-release:
    needs: [release, build]
    uses: JordanOlivet/ci-workflows/.github/workflows/gh-release.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      tag: ${{ needs.release.outputs.tag }}
    secrets: inherit
```

### Docker Application (docker-compose-manager)

```yaml
name: Release
on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  release:
    uses: JordanOlivet/ci-workflows/.github/workflows/reusable-release.yml@v1
    with:
      version-source: file
      version-file: VERSION
      changelog: true
    secrets: inherit

  build-backend:
    needs: release
    if: needs.release.outputs.should_release == 'true'
    uses: JordanOlivet/ci-workflows/.github/workflows/reusable-build-dotnet.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      project-path: docker-compose-manager-back
    secrets: inherit

  build-frontend:
    needs: release
    if: needs.release.outputs.should_release == 'true'
    uses: JordanOlivet/ci-workflows/.github/workflows/reusable-build-node.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      working-directory: docker-compose-manager-front
    secrets: inherit

  build-docker:
    needs: [release, build-backend, build-frontend]
    if: needs.release.outputs.should_release == 'true'
    uses: JordanOlivet/ci-workflows/.github/workflows/reusable-build-docker.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      image-name: docker-compose-manager
    secrets: inherit
```

## License

MIT
