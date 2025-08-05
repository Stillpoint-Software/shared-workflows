# 🌐 Shared Workflows Repository

This repository contains reusable **GitHub Actions workflows** for .NET projects.  
By centralizing the workflow logic here, all consumer repositories can use the same CI/CD standards without duplicating configuration.

---

## 📑 Table of Contents

- [🌐 Shared Workflows Repository](#-shared-workflows-repository)
  - [📑 Table of Contents](#-table-of-contents)
  - [📌 Available Workflows](#-available-workflows)
    - [1. ✅ Format](#1--format)
    - [2. 🧪 Test](#2--test)
        - [Prerequisites:](#prerequisites)
    - [3. 📊 Test Report](#3--test-report)
    - [4. 🌿 Create Issue Branch](#4--create-issue-branch)

---

## 📌 Available Workflows

### 1. ✅ Format

Automatically formats C# code using `dotnet format`.  
Commits any changes back to the repository.

**Consumer Usage**

```yaml
name: Format

on: 
  push:
  workflow_run: 
    workflows: 
      - Create Prerelease
      - Create Release
    types: [requested]
  workflow_dispatch:
  pull_request:
    types: [opened, edited, synchronize, reopened]
    branches:
      - main
      - develop

jobs:
  format:
    uses: Stillpoint-Software/shared-workflows/.github/workflows/format.yml@main
    with:
      dotnet_version: "9.0.x"
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
```


### 2. 🧪 Test

Builds and tests the solution.
Uploads .trx test results as artifacts.

##### Prerequisites:
Add a repository variable for the solution file:
1. Go to Respository Settings
2. Go to Actions → Variables
3. Enter the following:
  - Key: SOLUTION_NAME
  - Value: MyProject.sln

**Consumer Usage**
```yaml
name: Test

on: 
  workflow_run: 
    workflows: 
      - Create Prerelease
      - Create Release
    types: [requested]
  workflow_dispatch:
  pull_request:
    types: [opened, edited, synchronize, reopened]
    branches:
      - main
      - develop

jobs:
  test:
    uses: Stillpoint-Software/shared-workflows/.github/workflows/test.yml@main
    with:
      dotnet_version: "9.0.x"
      solution_name: ${{ vars.SOLUTION_NAME }}
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
```
### 3. 📊 Test Report

Generates a GitHub Checks report from test results.
Triggered after the Test workflow completes.

**Consumer Usage**

```yaml
name: Test Report
run-name: Generate Test Report for workflow ${{ github.event.workflow_run.name }} run ${{ github.event.workflow_run.run_number }} branch ${{ github.event.workflow_run.head_branch }}

on:
  workflow_run:
    workflows: [ "Test" ]
    types: 
      - completed

permissions:
  contents: read
  actions: read
  checks: write

jobs:
  report:
    uses: Stillpoint-Software/shared-workflows/.github/workflows/test-report.yml@main
    with:
      artifact_name: "test-results"
      test_report_name: "Unit Tests"
      path: "*.trx"
```

### 4. 🌿 Create Issue Branch
Automatically creates a new branch when issues or PRs are opened, assigned, or commented on.

**Consumer Usage**

```yaml
name: Create Issue Branch

on:
  issues:
    types: [ opened, assigned ]
  issue_comment:
    types: [ created ]
  pull_request:
    types: [ opened, closed ]

jobs:
  create_issue_branch_job:
    uses: Stillpoint-Software/shared-workflows/.github/workflows/create-issue-branch.yml@main
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

🔑 Why Shared Workflows?
- Centralized logic — update once, use everywhere
- Consumer flexibility — each repo controls when to run
- Consistency — all repos follow the same CI/CD rules
- Scalable — easy to add new workflows in one place

🚀 Quick Tips
- Set GH_TOKEN in your consumer repos to allow commits and PR updates.
- Use vars.SOLUTION_NAME instead of hardcoding solution names.
- You can chain workflows in a consumer repo by adding:
  
  ```yaml
  needs: format
  ```
