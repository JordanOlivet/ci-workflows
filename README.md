# ci-workflows

Centralized CI/CD workflows and composite actions for GitHub repositories.

## Overview

This repository provides reusable GitHub Actions workflows and composite actions for:

- **Release management**: Semantic versioning based on PR labels
- **Multi-platform builds**: Rust, .NET, Node.js
- **Docker builds**: Push to GitHub Container Registry
- **GitHub Releases**: Automated release creation with artifacts
- **Minecraft mods**: Gradle builds and Modrinth / CurseForge publishing

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

#### publish-mod.yml

Build a Gradle Minecraft mod at a release tag, publish one jar per loader to
Modrinth and CurseForge ([mc-publish](https://github.com/Kir-Antipov/mc-publish)),
and attach the jars to a GitHub Release. The tag gives the version
(`v0.1.0-alpha.1` builds with `-Pversion=0.1.0-alpha.1`); its core must match
`version=` in `gradle.properties`. The version type follows the pre-release
label (`alpha`, `beta`/`rc`, none: `release`).

```yaml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  publish:
    permissions:
      contents: write
    uses: JordanOlivet/ci-workflows/.github/workflows/publish-mod.yml@v1
    with:
      tag: ${{ github.ref_name }}
      mod-name: MyMod
      game-versions: 1.21.1
      modrinth-id: ${{ vars.MODRINTH_ID }}
      curseforge-id: ${{ vars.CURSEFORGE_ID }}
      targets: >-
        [{"loader":"neoforge","label":"NeoForge","files":"neoforge/build/libs/mymod-neoforge-*.jar"},
         {"loader":"fabric","label":"Fabric","files":"fabric/build/libs/mymod-fabric-*.jar"}]
    secrets:
      modrinth-token: ${{ secrets.MODRINTH_TOKEN }}
      curseforge-token: ${{ secrets.CURSEFORGE_TOKEN }}
```

**Inputs:** `tag`, `mod-name`, `targets` (JSON: `loader`, `label`, `files` glob
relative to `working-directory`), `game-versions` (required); `java-versions`
(`21`), `working-directory` (`.`), `version-file` (`gradle.properties`),
`java-version` (`21`), `modrinth-id` and `curseforge-id` (empty skips the
platform), `github-release` (`true`).

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
