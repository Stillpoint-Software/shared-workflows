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
  - [📌 Available Workflows and code to use them](#-available-workflows-and-code-to-use-them)
    - [1. 🏷️ Create Release](#1-️-create-release)
    - [2 📊 Create Test Report](#2--create-test-report)
    - [3. ✅ Format](#3--format)
    - [4. 🌿 Issue Branch](#4--issue-branch)
    - [5. 📦 Pack and Publish](#5--pack-and-publish)
    - [6. 🧪 Run Tests](#6--run-tests)
    - [7. Unlist Package](#7-unlist-package)
  - [🧩 Required Repo Setup](#-required-repo-setup)
  - [🚀 Quick Tips](#-quick-tips)


## 📌 Available Workflows and code to use them
---
### 1. 🏷️ Create Release
Creates/updates a GitHub Release with the specified version.  
Builds notes from `CHANGELOG.md` (falls back to commit list + compare link).

**Consumer Usage**
```yaml
name: Create Release

on:
  workflow_dispatch:
    inputs:
      mode:
        description: |
          How to set the version:
          - explicit: set to a specific value (e.g., 1.3-alpha)
          - bump: bump major/minor/patch from current SimpleVersion and apply optional prerelease
          - auto: policy-based (develop->next minor -alpha; hotfix/*->next patch -alpha; main/vX.Y->stable)
        type: choice
        options: [auto, bump, explicit]
        default: auto
      version:
        description: "When mode=explicit: exact version (e.g., 1.3-alpha, 1.2.3, 1.4.0-rc)"
        type: string
        default: ""
      increment:
        description: "When mode=bump: major | minor | patch"
        type: choice
        options: [major, minor, patch]
        default: patch
      prerelease:
        description: "When mode=bump: prerelease suffix WITHOUT leading dash (e.g., alpha, beta, rc). Leave blank for stable."
        type: string
        default: ""

permissions:
  contents: write
  pull-requests: write
  packages: write

run-name: "Create Release · ${{ inputs.mode }} · ${{ github.ref_name }}"

jobs:
  validate-inputs:
    runs-on: ubuntu-latest
    steps:
      - name: Check conditional requirements
        run: |
          set -euo pipefail
          MODE='${{ inputs.mode }}'
          VERSION='${{ inputs.version }}'
          INCR='${{ inputs.increment }}'
          if [[ "$MODE" == "explicit" && -z "$VERSION" ]]; then
            echo "? mode=explicit requires 'version' (e.g., 1.3-alpha)."; exit 1
          fi
          if [[ "$MODE" == "bump" && -z "$INCR" ]]; then
            echo "? mode=bump requires 'increment' (major|minor|patch)."; exit 1
          fi
          echo "? inputs look good."

  set-version:
    needs: validate-inputs
    uses: {{organization}}/shared-workflows/.github/workflows/set_version.yml@main
    with:
      target_branch: ${{ github.ref_name }}
      mode:         ${{ inputs.mode }}
      version:      ${{ inputs.version }}
      increment:    ${{ inputs.increment }}
      prerelease:   ${{ inputs.prerelease }}
    secrets: inherit

  create-release:
    needs: set-version
    uses: {{organization}}/shared-workflows/.github/workflows/prepare_release.yml@main
    with:
      target_branch: ${{ github.ref_name }}
      tag:          ${{ needs.set-version.outputs.tag }}
      prerelease:   ${{ needs.set-version.outputs.new_prerelease }}
      draft:        true
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

### 2 📊 Create Test Report
Turns uploaded `.trx` into a GitHub Checks report.

**Consumer Usage**
```yaml
name: Create Test Report

on:
  workflow_run:
    workflows: ["Run Tests"]
    types:     [completed]
    branches:  [main, develop]

permissions:
  contents: read
  actions:  read
  checks:   write

jobs:
  discover-auto:
    runs-on: ubuntu-latest
    if: ${{ github.event_name == 'workflow_run' }}
    outputs:
      branch_name: ${{ steps.meta.outputs.branch }}
      sha:         ${{ steps.meta.outputs.sha }}
      run_id:      ${{ steps.meta.outputs.run_id }}
      conclusion:  ${{ steps.meta.outputs.conclusion }}
    steps:
      - id: meta
        run: |
          echo "branch=${{ github.event.workflow_run.head_branch }}" >> "$GITHUB_OUTPUT"
          echo "sha=${{    github.event.workflow_run.head_sha    }}" >> "$GITHUB_OUTPUT"
          echo "run_id=${{ github.event.workflow_run.id }}"          >> "$GITHUB_OUTPUT"
          echo "conclusion=${{ github.event.workflow_run.conclusion }}" >> "$GITHUB_OUTPUT"

  report-auto:
    needs: discover-auto
    if: ${{ needs.discover-auto.outputs.conclusion != 'skipped' }}
    uses: {{organization}}/shared-workflows/.github/workflows/test_report.yml@main
    with:
      test_run_id: ${{ needs.discover-auto.outputs.run_id }}
      branch:      ${{ needs.discover-auto.outputs.branch_name }}
      sha:         ${{ needs.discover-auto.outputs.sha }}
    secrets: inherit
```

---

### 3. ✅ Format
Automatically formats C# with `dotnet format` and commits changes.

**Consumer Usage**
```yaml
name: Format

on:
  push:
  workflow_dispatch:
  pull_request:
    types: [opened, edited, synchronize, reopened]
    branches: [main, develop]

  workflow_run:
    workflows: [Create Prerelease, Create Release]
    types: [requested]
 
permissions:
  contents: write
  pull-requests: write
  actions: read                

jobs:
   discover:
    runs-on: ubuntu-latest
    outputs:
      branch_name: ${{ steps.set_branch.outputs.branch_name }}

    steps:
      - id: set_branch
        shell: bash
        run: |
          # 1. Pick the raw branch/ref for each trigger type
          if [[ "${{ github.event_name }}" == "workflow_run" ]]; then
            RAW='${{ github.event.workflow_run.head_branch }}'   
          elif [[ "${{ github.event_name }}" == "pull_request" ]]; then
            RAW='${{ github.event.pull_request.base.ref }}'       
          else                                                    
            RAW='${{ github.ref }}'                                
          fi

          # 2. Strip the refs/heads/ prefix if present
          CLEAN="${RAW#refs/heads/}"

          echo "Detected branch: $CLEAN"
          echo "branch_name=$CLEAN" >> "$GITHUB_OUTPUT"
          
   format:
    needs: discover
    if: ${{ needs.discover.result == 'success' }}
    uses: {{organization}}/shared-workflows/.github/workflows/format.yml@main
    with:
      branch: ${{ needs.discover.outputs.branch_name }}
    secrets: inherit
```

---
### 4. 🌿 Issue Branch
Creates a branch when issues/PRs are opened or assigned.

**Consumer Usage**
```yaml
name: Create Issue Branch

on:
  issues:
    types: [opened, assigned]        # “immediate” / “auto” modes
  issue_comment:
    types: [created]                 # ChatOps mode
  pull_request:
    types: [opened, closed]          # PR-related features

permissions:
  contents: read
  issues: write
  pull-requests: write

jobs:
  discover:
    runs-on: ubuntu-latest
    outputs:
      branch_name: ${{ steps.set_branch.outputs.branch_name }}
    steps:
      - name: Determine target branch
        id: set_branch
        run: echo "branch_name=${BRANCH}" >> "$GITHUB_OUTPUT"
        env:
          BRANCH: >-
            ${{ github.event_name == 'pull_request' &&
                github.event.pull_request.base.ref ||
                'main' }}

  create-issue-branch-main:
    needs: discover
    if: ${{ needs.discover.result == 'success' }}
    uses: {{organization}}/shared-workflows/.github/workflows/issue_branch.yml@main
    secrets: inherit
```

### 5. 📦 Pack and Publish
Restores, builds, tests, then packs and pushes to NuGet.  
For **stable** builds, sets `PublicRelease=true`.


**Consumer Usage (triggered on Release published)**
```yaml
﻿name: Pack and Publish

on:
  release:
    types: [published]

permissions:
  contents: write
  pull-requests: write
  packages: write
  statuses: write

jobs:
  set_config:
    uses: {{organization}}/shared-workflows/.github/workflows/determine_build_configuration.yml@main
    with:
      trigger: release
      target_branch: ${{ github.event.release.target_commitish }}
      override_build_configuration: ''                 
      prerelease: ${{ github.event.release.prerelease }} # true/false from the release
  
  tests:
    needs: set_config
    uses: {{organization}}/shared-workflows/.github/workflows/run_tests.yml@main
    with:
      branch: ${{ github.event.release.target_commitish }}
      solution_name: ${{ vars.SOLUTION_NAME }}

  publish:
    needs: [set_config, tests]
    uses: {{organization}}/shared-workflows/.github/workflows/pack_and_publish.yml@main
    with:
      build_configuration: ${{ needs.set_config.outputs.build_configuration }}
    secrets:
      NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}

  result:
    needs: [publish, tests]
    if: always()
    runs-on: ubuntu-latest
    steps:
       - run: echo "Tests result = ${{ needs.tests.result }}"
       - run: echo "Pack & Publish result = ${{ needs.publish.result }}"
```
---
### 6. 🧪 Run Tests
Builds and tests the solution for .NET 8, 9, and 10.

**Consumer Usage**
```yaml
name: Run Tests

on:
  workflow_run:
    workflows: [Create Release]
    types: [requested]
    branches: [main, develop]        
  workflow_dispatch:
  pull_request:
    types: [opened, edited, synchronize, reopened]
    branches: [main, develop]

permissions:
  contents: read
  actions: read

jobs:
  discover:
    runs-on: ubuntu-latest
    outputs:
      branch_name: ${{ steps.set_branch.outputs.branch_name }}

    steps:
      - id: set_branch
        shell: bash
        run: |
          # 1. Pick the raw branch/ref for each trigger type
          if [[ "${{ github.event_name }}" == "workflow_run" ]]; then
            RAW='${{ github.event.workflow_run.head_branch }}'   
          elif [[ "${{ github.event_name }}" == "pull_request" ]]; then
            RAW='${{ github.event.pull_request.base.ref }}'       
          else                                                    
            RAW='${{ github.ref }}'                                
          fi

          # 2. Strip the refs/heads/ prefix if present
          CLEAN="${RAW#refs/heads/}"

          echo "Detected branch: $CLEAN"
          echo "branch_name=$CLEAN" >> "$GITHUB_OUTPUT"

  test:
    needs: discover
    uses: {{organization}}/shared-workflows/.github/workflows/run_tests.yml@main
    with:
      branch: ${{ needs.discover.outputs.branch_name }}
      solution_name: ${{ vars.SOLUTION_NAME }}
    secrets: inherit
```

---
### 7. Unlist Package
Unlists a specific version of a NuGet package from nuget.org.

**Consumer Usage**  
```yaml
name: Unlist NuGet Package

on:
  workflow_dispatch:
    inputs:
      package_id:
        description: "NuGet package ID (e.g., Hyperbee.xxx)"
        required: true
        type: string
      package_version:
        description: "Exact version (e.g., v1.2.1-alpha-ge4caaff67a)"
        required: true
        type: string
      dry_run:
        description: "If true, shows what would happen"
        required: false
        default: false
        type: boolean

permissions:
  contents: read

run-name: "Unlist · ${{ inputs.package_id }} · ${{ inputs.package_nuget }}"

jobs:
  unlist:
    uses: {{organization}}/shared-workflows/.github/workflows/unlist-package.yml@main
    secrets: inherit
    with:
      package_id:      ${{ inputs.package_id }}
      package_version: ${{ inputs.package_version }}
      dry_run:         ${{ inputs.dry_run }}
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
    <PackageReference Include="Nerdbank.GitVersioning" Version="3.9.50" PrivateAssets="all" />
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

  <!-- Global project properties - .NET 10 LTS First Strategy -->
  <PropertyGroup>
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- Primary target: .NET 10 (next LTS), with .NET 9 for current support, .NET 8 for compatibility -->
    <TargetFrameworks>net10.0;net9.0;net8.0</TargetFrameworks>
  </PropertyGroup>
</Project>
```

---

## 🚀 Quick Tips
- The branches MUST be main, develop, hotfix/*, and release/vX.Y for versioning to work correctly.
- Use `${{ secrets.GITHUB_TOKEN }}` as **GH_TOKEN** in consumers.
- `create_release.yml` **pushes** `version.json`. Ensure branch protections allow workflow pushes (or switch it to open a PR).
- Tag is always `v<NuGetPackageVersion>`. Releases are created as **draft** by default.
- For prereleases (develop/hotfix),  **Nerdbank.GitVersioning** baseline is `-alpha`; `main` is **always stable**.
- NuGet push uses `--skip-duplicate` to keep reruns green.
- The test will run for 8,9,10.
