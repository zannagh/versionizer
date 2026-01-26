# versionizer

Reusable GitHub Actions and build tooling for consistent semantic versioning across Java/Kotlin and .NET projects. Built on [GitVersion](https://gitversion.net/).

## Actions

All actions are composite and can be referenced as:

```yaml
uses: zannagh/versionizer/<action>@v1
```

### `calculate-version`

Sets up GitVersion and calculates semantic version properties from git history. Ships with built-in default configs so you don't need a `GitVersion.yml` in your repo.

**Config resolution priority:**
1. `config-file` input (explicit override)
2. Repo-level `GitVersion.yml` / `GitVersion.yaml` (auto-discovered)
3. Bundled default for the chosen `strategy`

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `gitversion-version` | `6.x` | GitVersion version spec to install |
| `strategy` | `continuous-deployment` | Default strategy when no `GitVersion.yml` exists in the repo. Options: `continuous-deployment` (every commit to main gets a prerelease, tags produce stable versions) or `continuous-delivery` (.NET-friendly, manual release gating, assembly versioning formats) |
| `config-file` | `''` | Path to a custom GitVersion config file. Overrides both repo config and strategy default |

**Outputs:**

| Output | Description |
|--------|-------------|
| `majorMinorPatch` | `Major.Minor.Patch` string (e.g. `1.2.3`) |
| `fullSemVer` | Full semantic version with prerelease info |
| `semVer` | Semantic version |
| `major` / `minor` / `patch` | Individual version components |
| `preReleaseLabel` | Prerelease label (e.g. `pre`, `beta`) |
| `preReleaseNumber` | Prerelease number |
| `commitsSinceVersionSource` | Commits since the last version tag |
| `assemblySemVer` | Assembly version (for .NET) |
| `assemblySemFileVer` | Assembly file version (for .NET) |
| `informationalVersion` | Informational version (for .NET) |
| `branchName` / `sha` / `shortSha` | Git context |
| `configSource` | Where the config came from: `repo`, `default`, or `custom` |

**Usage:**

```yaml
# Minimal — uses bundled continuous-deployment config if no GitVersion.yml in repo
- uses: actions/checkout@v6
  with:
    fetch-depth: 0  # Required for GitVersion

- uses: zannagh/versionizer/calculate-version@v1
  id: version

- run: echo "Version is ${{ steps.version.outputs.fullSemVer }}"
```

```yaml
# .NET project — use the continuous-delivery strategy
- uses: zannagh/versionizer/calculate-version@v1
  id: version
  with:
    strategy: continuous-delivery
```

```yaml
# Inject a custom config file
- uses: zannagh/versionizer/calculate-version@v1
  id: version
  with:
    config-file: ./my-custom-gitversion.yml
```

---

### `check-release`

Determines whether a release should be created. Checks the GitHub event type, commit message conventions, and changed files.

Commits prefixed with `ci:`, `docs:`, `build:`, or `chore:` are considered non-code and skip releases. Only files that look like actual source code trigger a release (documentation, config, and CI files are excluded).

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `event-name` | *required* | `github.event_name` |
| `is-dry-run` | `false` | Dry-run mode (still proceeds with release) |
| `force-release` | `false` | Force release regardless of changes |

**Outputs:**

| Output | Description |
|--------|-------------|
| `should_release` | `true` if a release should be created |
| `is_manual_release` | `true` if triggered from a GitHub Release event |
| `skip_reason` | Why the release was skipped (empty if not) |

**Usage:**

```yaml
- uses: zannagh/versionizer/check-release@v1
  id: release-check
  with:
    event-name: ${{ github.event_name }}
    is-dry-run: ${{ inputs.dry_run }}

- name: Build
  if: steps.release-check.outputs.should_release == 'true'
  run: ./build.sh
```

---

### `determine-version`

Computes a final version string from GitVersion outputs and release context. For manual releases (GitHub Release events), extracts the version from the tag. For automatic releases, formats as `{semver}-{prefix}.{number}`.

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `is-manual-release` | *required* | From `check-release` output |
| `release-tag-name` | `''` | GitHub release tag (e.g. `v1.2.3`) |
| `release-prerelease` | `false` | Whether the GitHub release is a prerelease |
| `semver` | `''` | `majorMinorPatch` from `calculate-version` |
| `prerelease-number` | `''` | `preReleaseNumber` from `calculate-version` |
| `commits-since` | `''` | `commitsSinceVersionSource` from `calculate-version` |
| `prerelease-prefix` | `pre` | Prerelease label (`pre`, `beta`, `rc`, etc.) |

**Outputs:**

| Output | Description |
|--------|-------------|
| `version` | The computed version string (e.g. `1.2.3` or `1.2.3-pre.5`) |
| `is_prerelease` | `true` if this is a prerelease build |

**Usage:**

```yaml
- uses: zannagh/versionizer/determine-version@v1
  id: mod-ver
  with:
    is-manual-release: ${{ steps.release-check.outputs.is_manual_release }}
    release-tag-name: ${{ github.event.release.tag_name }}
    semver: ${{ steps.version.outputs.majorMinorPatch }}
    prerelease-number: ${{ steps.version.outputs.preReleaseNumber }}
    commits-since: ${{ steps.version.outputs.commitsSinceVersionSource }}
    prerelease-prefix: beta  # optional, defaults to "pre"
```

---

### `setup-gradle-env`

Sets up a Java/Kotlin Gradle build environment: validates the Gradle wrapper, installs JDK, configures Gradle, and makes `gradlew` executable.

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `java-version` | `25` | JDK version |
| `java-distribution` | `temurin` | JDK distribution (`temurin`, `microsoft`, `corretto`, etc.) |

**Usage:**

```yaml
- uses: actions/checkout@v6

- uses: zannagh/versionizer/setup-gradle-env@v1
  with:
    java-version: '21'

- run: ./gradlew build
```

---

### `setup-dotnet-env`

Sets up the .NET SDK with optional NuGet private feed authentication and GitVersion CLI tooling.

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `dotnet-version` | `9.0.x` | .NET SDK version |
| `nuget-source-url` | `''` | Private NuGet feed URL (e.g. GitHub Packages) |
| `nuget-auth-token` | `''` | Auth token for the private feed |
| `install-gitversion-tool` | `true` | Install `gitversion.tool` globally |

**Usage:**

```yaml
- uses: actions/checkout@v6
  with:
    fetch-depth: 0

- uses: zannagh/versionizer/setup-dotnet-env@v1
  with:
    dotnet-version: '9.0.x'
    nuget-source-url: https://nuget.pkg.github.com/my-org/index.json
    nuget-auth-token: ${{ secrets.PACKAGES_READ_PAT }}
```

---

## MSBuild Targets (Local .NET Versioning)

`msbuild/GlobalVersioning.targets` provides automatic GitVersion-based versioning for local .NET development.

### How It Works

1. **Debug builds** skip GitVersion entirely and use `0.0.1-dev` — fast iteration, no git dependency.
2. **Release builds** run `dotnet gitversion`, cache the JSON output in `.gitversion.done`, and reuse it until the Git HEAD changes.
3. GitVersion tool is auto-installed globally if not found.

### Setup

Add to your `Directory.Build.props`:

```xml
<Project>
  <Import Project="build/GlobalVersioning.targets" />

  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <!-- ... -->
  </PropertyGroup>
</Project>
```

Copy `msbuild/GlobalVersioning.targets` into your project's `build/` directory, and add `.gitversion.done` to your `.gitignore`.

### Properties Set

| MSBuild Property | Source |
|-----------------|--------|
| `Version` | `FullSemVer` |
| `AssemblyVersion` | `AssemblySemVer` |
| `FileVersion` | `AssemblySemFileVer` |
| `InformationalVersion` | `InformationalVersion` |
| `ApplicationVersion` | `AssemblySemFileVer` (for ClickOnce) |
| `ApplicationRevision` | `0` |

---

## Example GitVersion Configurations

Two example `GitVersion.yml` files are provided under `examples/`:

### ContinuousDeployment (`examples/gitversion-continuous-deployment.yml`)

Best for projects that auto-deploy from main. Every commit to main gets a unique prerelease version (e.g. `1.0.1-pre.5`). Tagged commits produce stable versions.

- Version bumps via merge commit messages: `+semver: major`, `+semver: minor`, `+semver: patch`
- Only merge messages trigger increments (`MergeMessageOnly`)
- Prerelease label: `pre`

### ContinuousDelivery (`examples/gitversion-continuous-delivery.yml`)

Best for .NET projects with manual release gating. Main branch produces stable versions when tagged. Includes .NET-specific assembly versioning formats.

- PRs get `ci` label, unknown branches get `preview`
- Assembly versioning: `Major.Minor.Patch.WeightedPreReleaseNumber`
- Ignores `GitVersion.yml` changes in version calculation

---

## Full Workflow Examples

### Java/Kotlin (Gradle)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: zannagh/versionizer/setup-gradle-env@v1

      - uses: zannagh/versionizer/calculate-version@v1
        id: version

      - uses: zannagh/versionizer/check-release@v1
        id: release-check
        with:
          event-name: ${{ github.event_name }}

      - uses: zannagh/versionizer/determine-version@v1
        id: mod-ver
        with:
          is-manual-release: ${{ steps.release-check.outputs.is_manual_release }}
          release-tag-name: ${{ github.event.release.tag_name }}
          release-prerelease: ${{ github.event.release.prerelease }}
          semver: ${{ steps.version.outputs.majorMinorPatch }}
          prerelease-number: ${{ steps.version.outputs.preReleaseNumber }}
          commits-since: ${{ steps.version.outputs.commitsSinceVersionSource }}

      - run: |
          ./gradlew build \
            -PsemVer="${{ steps.mod-ver.outputs.version }}" \
            -Pprerelease="${{ steps.mod-ver.outputs.is_prerelease }}"
```

### .NET

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: zannagh/versionizer/setup-dotnet-env@v1
        with:
          nuget-source-url: https://nuget.pkg.github.com/my-org/index.json
          nuget-auth-token: ${{ secrets.PACKAGES_READ_PAT }}

      - run: dotnet build --configuration Release
      # Version is automatically set by GlobalVersioning.targets
      # (imported in Directory.Build.props)
```
