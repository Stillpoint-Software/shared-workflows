# 🌐 Shared Workflows Repository

This repository contains reusable **GitHub Actions workflows** for .NET projects.  
Update once here → use everywhere.

**NOTE:** If the **Main** repo has restrictions that a pull request is required to 
update branch, then the **Create Release** workflow will fail.  

You will need to use the **Create Release** workflow on the *develop* branch and then create a pull request to main.

---

## 📑 Table of Contents
- [🌐 Shared Workflows Repository](#-shared-workflows-repository)
  - [📑 Table of Contents](#-table-of-contents)
  - [📌 Available Workflows](#-available-workflows)
    - [1. ✅ Format](#1--format)
    - [2. 🧪 Run Tests](#2--run-tests)
    - [3. 📊 Create Test Report](#3--create-test-report)
    - [4. 🌿 Issue Branch](#4--issue-branch)
    - [5. 🔢 Set Version](#5--set-version)
    - [6. 🏷️ Prepare Release (draft + changelog)](#6-️-prepare-release-draft--changelog)
    - [7. 📦 Pack and Publish](#7--pack-and-publish)
    - [8. ⚙️ Determine Build Configuration](#8-️-determine-build-configuration)
  - [🧩 Required Repo Setup](#-required-repo-setup)
  - [🚀 Quick Tips](#-quick-tips)

---

## 📌 Available Workflows

### 1. ✅ Format
Automatically formats C# with `dotnet format` and commits changes.

**Consumer Usage**
```yaml
name: Format
on:
  push:
  workflow_run:
    workflows: [ Create Prerelease, Create Release ]
    types: [requested]
  workflow_dispatch:
  pull_request:
    types: [opened, edited, synchronize, reopened]
    branches: [ main, develop ]

jobs:
  format:
    uses: Github Organization Name/shared-workflows/.github/workflows/format.yml@main
    with:
      dotnet_version: "9.0.x"
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

### 2. 🧪 Run Tests
Builds and tests the solution. Uploads `.trx` as artifacts.

**Consumer Usage**
```yaml
name:Run Tests
on:
  workflow_run:
    workflows: [ Create Prerelease, Create Release ]
    types: [requested]
  workflow_dispatch:
  pull_request:
    types: [opened, edited, synchronize, reopened]
    branches: [ main, develop ]

jobs:
  test:
    uses: Github Organization Name/shared-workflows/.github/workflows/run_tests.yml@main
    with:
      dotnet_version: "9.0.x"
      solution_name: ${{ vars.SOLUTION_NAME }}
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

### 3. 📊 Create Test Report
Turns uploaded `.trx` into a GitHub Checks report.

**Consumer Usage**
```yaml
name: Test Report
run-name: "Generate Test Report for ${{ github.event.workflow_run.name }} #${{ github.event.workflow_run.run_number }} on ${{ github.event.workflow_run.head_branch }}"
on:
  workflow_run:
    workflows: [ "Test" ]
    types: [ completed ]

permissions:
  contents: read
  actions: read
  checks: write

jobs:
  report:
    uses: Github Organization Name/shared-workflows/.github/workflows/test_report.yml@main
    with:
      artifact_name: "test-results"
      test_report_name: "Unit Tests"
      path: "*.trx"
```

---

### 4. 🌿 Issue Branch
Creates a branch when issues/PRs are opened or assigned.

**Consumer Usage**
```yaml
name: Issue Branch
on:
  issues:        { types: [ opened, assigned ] }
  issue_comment: { types: [ created ] }
  pull_request:  { types: [ opened, closed ] }

jobs:
  create_issue_branch_job:
    uses: Github Organization Name/shared-workflows/.github/workflows/issue_branch.yml@main
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

### 5. 🔢 Set Version 
Computes & sets the version via **Nerdbank.GitVersioning** and **pushes** `version.json`.  
Guarantees the **version always increases** and the **tag = v\<NuGetPackageVersion\>** is unique (auto-bumps if needed).

**Reusable:** `.github/workflows/set_version.yml`  
**Inputs:** `target_branch`, `mode` (`auto|bump|explicit`), optional `version`, `increment`, `prerelease`  
**Outputs:** `tag`, `new_simple`, `new_prerelease`, `new_nuget`

**Consumer Usage**
```yaml
jobs:
  set-version:
    uses: Github Organization Name/shared-workflows/.github/workflows/set_version.yml@main
    with:
      target_branch: ${{ github.ref_name }}
      mode: auto              
      version: ""             
      increment: patch        
      prerelease: ""          
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

### 6. 🏷️ Prepare Release (draft + changelog)
Creates/updates a **draft** GitHub Release for the provided tag.  
Builds notes from `CHANGELOG.md` (falls back to commit list + compare link).

**Reusable:** `.github/workflows/prepare_release.yml`  
**Inputs:** `target_branch`, `tag`, optional `prerelease`, `draft` (default true)

**Consumer Usage**
```yaml
jobs:
  prepare-release:
    needs: set-version
    uses: Github Organization Name/shared-workflows/.github/workflows/prepare_release.yml@main
    with:
      target_branch: ${{ github.ref_name }}
      tag:        ${{ needs.set-version.outputs.tag }}
      prerelease: ${{ needs.set-version.outputs.new_prerelease }}
      draft:      true
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

### 7. 📦 Pack and Publish
Restores, builds, tests, then packs and pushes to NuGet.  
For **stable** builds, sets `PublicRelease=true`.

**Reusable:** `.github/workflows/pack_and_publish.yml`  
**Inputs:** `build_configuration` (`Release|Debug`)

**Consumer Usage (triggered on Release published)**
```yaml
name: Publish
on:
  release:
    types: [ published ]

permissions:
  contents: write
  pull-requests: write
  packages: write

jobs:
  set-config:
    uses: Github Organization Name/shared-workflows/.github/workflows/determine_build_configuration.yml@main
    with:
      trigger: release
      target_branch: ${{ github.event.release.target_commitish }}
      override_build_configuration: '' 
      prerelease: ${{ github.event.release.prerelease }}  

  publish:
    needs: set-config
    uses: Github Organization Name/shared-workflows/.github/workflows/dotnet_pack.yml@main
    with:
      build_configuration: ${{ needs.set-config.outputs.build_configuration }}
    secrets:
      NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
```

---

### 8. ⚙️ Determine Build Configuration
Resolves `Release` vs `Debug` for downstream jobs.  
Release events → `prerelease=false` ⇒ `Release`, else `Debug`.  
Manual runs can pass an override.

**Reusable:** `.github/workflows/determine_build_configuration.yml`  
**Inputs:** `trigger`, `target_branch`, `override_build_configuration`, `prerelease`  
**Outputs:** `build_configuration`

**Consumer Usage (example for manual CI)**
```yaml
jobs:
  set-config:
    uses: Github Organization Name/shared-workflows/.github/workflows/determine_build_configuration.yml@main
    with:
      trigger: workflow_dispatch
      target_branch: ${{ github.ref_name }}
      override_build_configuration: 'Debug' 
      prerelease: false
```
---

## 🧩 Required Repo Setup

1) **Repository variable**
   - `SOLUTION_NAME` → e.g., `MyProject.sln`

2) **Versioning: `version.json`**
   - Root `version.json` managed by NBGV (Nerdbank GitVersioning) (baseline rules in your repo).
  
  ```json
  {
  "$schema": "https://raw.githubusercontent.com/dotnet/Nerdbank.GitVersioning/main/src/NerdBank.GitVersioning/version.schema.json",
  "version": "1.0.0",
  "publicReleaseRefSpec": [
    "^refs/heads/main$",
    "^refs/heads/hotfix$",
    "^refs/heads/v\\d+\\.\\d+$"
  ]
}
```


3) **MSBuild props (at repo root)**
   - File must be named **`Directory.Build.props`**.
   - Minimal recommended content:
```xml
<Project>
  <!-- Shared package refs -->
  <ItemGroup>
    <!-- NBGV drives versions; PrivateAssets=all keeps it out of consumers -->
    <PackageReference Include="Nerdbank.GitVersioning" Version="3.8.38-alpha" PrivateAssets="all" />
    <!-- SourceLink for GitHub -->
    <PackageReference Include="Microsoft.SourceLink.GitHub" Version="8.0.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>

  <!-- SourceLink / build hygiene -->
  <PropertyGroup>
    <!-- Deterministic builds + embed sources for better debugging -->
    <ContinuousIntegrationBuild>true</ContinuousIntegrationBuild>
    <Deterministic>true</Deterministic>
    <EmbedUntrackedSources>true</EmbedUntrackedSources>
    <PublishRepositoryUrl>true</PublishRepositoryUrl>
  </PropertyGroup>

  <!-- Package metadata applied only to packable projects -->
  <PropertyGroup Condition="'$(IsPackable)' == 'true'">
    <!-- NuGet README -->
    <PackageReadmeFile>README.md</PackageReadmeFile>

    <!-- NuGet Release Notes link -->
    <PackageReleaseNotes>https://github.com/Stillpoint-Software/Hyperbee.XS/releases/latest</PackageReleaseNotes>

    <!-- Repository metadata (shows on NuGet) -->
    <RepositoryUrl>https://github.com/Stillpoint-Software/Hyperbee</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
    <PackageProjectUrl>https://github.com/Stillpoint-Software/Hyperbee</PackageProjectUrl>
  </PropertyGroup>

  <!-- Pull the root README into the package root -->
  <ItemGroup Condition="'$(IsPackable)' == 'true'">
    <None Include="$(MSBuildThisFileDirectory)README.md"
          Pack="true"
          PackagePath="\"
          Link="README.md" />
  </ItemGroup>
</Project>
```

---

## 🚀 Quick Tips
- Use `${{ secrets.GITHUB_TOKEN }}` as **GH_TOKEN** in consumers.
- `set_version.yml` **pushes** `version.json`. Ensure branch protections allow workflow pushes (or switch it to open a PR).
- Tag is always `v<NuGetPackageVersion>`. Releases are created as **draft** by default.
- For prereleases (develop/hotfix),  **Nerdbank.GitVersioning** baseline is `-alpha`; `main` is **always stable**.
- NuGet push uses `--skip-duplicate` to keep reruns green.