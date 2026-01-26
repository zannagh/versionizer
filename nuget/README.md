# Versionizer

Automatic GitVersion-based semantic versioning for .NET projects.

Works in **any** build environment — `dotnet build`, Visual Studio, Rider, VSCode — because it uses the GitVersion CLI tool via `Exec` rather than a managed MSBuild task assembly (which is what causes `GitVersion.MsBuild` to break on newer VS versions).

## Quick Start

```xml
<PackageReference Include="Versionizer" Version="1.0.0" PrivateAssets="all" />
```

That's it. On `dotnet build --configuration Release`, your project gets versioned automatically from git history.

## How It Works

1. **Debug builds** skip GitVersion entirely and use `0.0.1-dev` — no git overhead during development.
2. **Release builds** run `dotnet gitversion`, cache the JSON result in `.gitversion.done` (keyed by HEAD SHA), and reuse it across all projects in the solution until HEAD changes.
3. **gitversion.tool** is auto-installed globally if not found.

Add `.gitversion.done` to your `.gitignore`.

## Properties Set

| Property | GitVersion Source |
|----------|-------------------|
| `Version` | `FullSemVer` |
| `AssemblyVersion` | `AssemblySemVer` |
| `FileVersion` | `AssemblySemFileVer` |
| `InformationalVersion` | `InformationalVersion` |
| `ApplicationVersion` | `AssemblySemFileVer` (for ClickOnce) |

## Configuration

| MSBuild Property | Default | Description |
|-----------------|---------|-------------|
| `VersionizerEnabled` | `true` | Set to `false` to disable for a specific project |
| `VersionizerDebugVersion` | `0.0.1-dev` | Version string used in Debug builds |
| `RepositoryRoot` | *(auto-detected)* | Override if auto-detection fails |

Set these in your `.csproj` or `Directory.Build.props`:

```xml
<PropertyGroup>
  <!-- Use a different debug version -->
  <VersionizerDebugVersion>0.0.0-local</VersionizerDebugVersion>
</PropertyGroup>
```

To disable for a specific project:

```xml
<PropertyGroup>
  <VersionizerEnabled>false</VersionizerEnabled>
</PropertyGroup>
```

## Requirements

- Git repository with tags for version history
- A `GitVersion.yml` in the repository root (see [examples](https://github.com/zannagh/versionizer/tree/main/examples))
- .NET SDK (for `dotnet tool` to install GitVersion CLI)
