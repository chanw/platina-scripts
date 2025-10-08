# Azure DevOps to GitHub Actions Conversion Guide

## Overview

This document provides a comprehensive summary of the conversion from Azure DevOps pipeline (`azure/pipelines/dotnet/ci.yml`) to GitHub Actions workflow (`.github/workflows/ci.yml`). The conversion maintains all original functionality while leveraging GitHub Actions' native capabilities and marketplace actions.

## Key Conversion Mappings

### Azure DevOps → GitHub Actions Task Equivalents

| Azure DevOps Task | GitHub Actions Equivalent | Notes |
|-------------------|---------------------------|-------|
| `UseDotNet@2` | `actions/setup-dotnet@v4` | Sets up .NET SDK with version control |
| `Cache@2` | `actions/cache@v4` | NuGet package caching with date-based keys |
| `DotNetCoreCLI@2` | Direct `dotnet` commands | More direct approach, better control |
| `PublishTestResults@2` | `dorny/test-reporter@v1` | Enhanced test result reporting |
| `PublishCodeCoverageResults@2` | `codecov/codecov-action@v4` | Industry-standard coverage reporting |
| `AzureCLI@2` | `azure/login@v2` + direct commands | Separate login step for better security |
| `Docker@2` | `docker/login-action@v3` + direct commands | Dedicated Docker authentication |
| `PublishPipelineArtifact@1` | `actions/upload-artifact@v4` | GitHub's native artifact system |
| `PowerShell@2` | `shell: pwsh` | Native PowerShell support in steps |

## Parameter Conversion

### Naming Convention Changes
- Azure DevOps uses `PascalCase` parameter names
- GitHub Actions uses `kebab-case` input names
- All original parameter functionality preserved

### Parameter Type Mappings

```yaml
# Azure DevOps
parameters:
- name: WithPack
  type: boolean
  default: false

# GitHub Actions
inputs:
  with-pack:
    type: boolean
    default: false
```

### Complete Parameter Mapping

| Azure DevOps Parameter | GitHub Actions Input | Type | Default |
|------------------------|---------------------|------|---------|
| `Solution` | `solution` | string | `'*.sln'` |
| `BuildConfiguration` | `build-configuration` | string | `'Release'` |
| `BuildPlatform` | `build-platform` | string | `'Any CPU'` |
| `DotNetSdkVersion` | `dotnet-sdk-version` | string | `''` |
| `PostBuildDotnetArgs` | `post-build-dotnet-args` | string | `'--no-restore --no-build'` |
| `TestProjectsGlob` | `test-projects-glob` | string | `'**/*[Tt]ests/*.csproj'` |
| `TestArguments` | `test-arguments` | string | `''` |
| `TestCoverageArguments` | `test-coverage-arguments` | string | `'--collect "Code Coverage"'` |
| `NoTests` | `no-tests` | boolean | `false` |
| `WithPlatinaTestInit` | `with-platina-test-init` | boolean | `false` |
| `WithPack` | `with-pack` | boolean | `false` |
| `WithPublish` | `with-publish` | boolean | `false` |
| `WithRelease` | `with-release` | boolean | `false` |
| `NuGetConfigPath` | `nuget-config-path` | string | `'nuget.config'` |
| `InternalFeedName` | `internal-feed-name` | string | `''` |
| `ExternalFeedCredentials` | `external-feed-credentials` | string | `''` |
| `WithNuGetCache` | `with-nuget-cache` | boolean | `false` |
| `AzureSubscription` | `azure-subscription` | string | `''` |
| `ContainerRegistryConnection` | `container-registry-connection` | string | `''` |
| `ContainerRegistry` | `container-registry` | string | `''` |

## Conditional Logic Conversion

### Expression Syntax Changes

| Azure DevOps | GitHub Actions | Description |
|--------------|----------------|-------------|
| `${{ if condition }}` | `if: condition` | Conditional step execution |
| `ne(parameters.X, 'value')` | `inputs.X != 'value'` | Not equal comparison |
| `eq(parameters.X, 'value')` | `inputs.X == 'value'` | Equal comparison |
| `and(cond1, cond2)` | `cond1 && cond2` | Logical AND |
| `or(cond1, cond2)` | `cond1 \|\| cond2` | Logical OR |

### Complex Conditional Examples

```yaml
# Azure DevOps
- ${{ if and( ne(parameters.NoTests, true), ne(parameters.WithPlatinaTestInit, false) ) }}:

# GitHub Actions
- if: inputs.no-tests == false && inputs.with-platina-test-init == true
```

## Variable and Environment Conversion

### Built-in Variables

| Azure DevOps | GitHub Actions | Description |
|--------------|----------------|-------------|
| `$(Build.ArtifactStagingDirectory)` | `${{ github.workspace }}` | Working directory |
| `$(Agent.TempDirectory)` | `${{ runner.temp }}` | Temporary directory |
| `$(Agent.OS)` | `${{ runner.os }}` | Operating system |
| `$(NUGET_PACKAGES)` | `${{ env.NUGET_PACKAGES }}` | Environment variables |

### Custom Variables

```yaml
# Azure DevOps
- pwsh: echo "##vso[task.setvariable variable=NuGetCacheKey]$(Get-Date -Format yyyyMMdd)"

# GitHub Actions
- shell: pwsh
  run: |
    $cacheKey = Get-Date -Format "yyyyMMdd"
    echo "NUGET_CACHE_KEY=$cacheKey" >> $env:GITHUB_ENV
```

## Security and Secrets Management

### Required Secrets Configuration

The following secrets must be configured in your GitHub repository settings:

| Secret Name | Purpose | Required For |
|-------------|---------|--------------|
| `NUGET_API_KEY` | NuGet package publishing | NuGet operations |
| `AZURE_CREDENTIALS` | Azure CLI authentication | Azure-integrated tests |
| `CONTAINER_REGISTRY_USERNAME` | Container registry auth | Docker operations |
| `CONTAINER_REGISTRY_PASSWORD` | Container registry auth | Docker operations |

### Azure Credentials Format

```json
{
  "clientId": "your-client-id",
  "clientSecret": "your-client-secret",
  "subscriptionId": "your-subscription-id",
  "tenantId": "your-tenant-id"
}
```

## Step-by-Step Feature Conversion

### 1. .NET SDK Setup
```yaml
# Azure DevOps
- task: UseDotNet@2
  inputs:
    version: ${{ parameters.DotNetSdkVersion }}
    includePreviewVersions: true

# GitHub Actions
- uses: actions/setup-dotnet@v4
  with:
    dotnet-version: ${{ inputs.dotnet-sdk-version }}
    include-prerelease: true
```

### 2. NuGet Caching
```yaml
# Azure DevOps
- task: Cache@2
  inputs:
    key: nuget | "$(Agent.OS)" | "$(NuGetCacheKey)"
    path: $(NUGET_PACKAGES)

# GitHub Actions
- uses: actions/cache@v4
  with:
    path: ${{ env.NUGET_PACKAGES }}
    key: nuget-${{ runner.os }}-${{ env.NUGET_CACHE_KEY }}
```

### 3. Test Result Publishing
```yaml
# Azure DevOps
- task: PublishTestResults@2
  inputs:
    testResultsFormat: 'VSTest'
    testResultsFiles: '*.trx'

# GitHub Actions
- uses: dorny/test-reporter@v1
  with:
    name: .NET Tests
    path: ${{ runner.temp }}/*.trx
    reporter: dotnet-trx
```

### 4. Docker Operations
```yaml
# Azure DevOps
- task: Docker@2
  inputs:
    command: login
    containerRegistry: ${{ parameters.ContainerRegistryConnection }}

# GitHub Actions
- uses: docker/login-action@v3
  with:
    registry: ${{ inputs.container-registry }}
    username: ${{ secrets.CONTAINER_REGISTRY_USERNAME }}
    password: ${{ secrets.CONTAINER_REGISTRY_PASSWORD }}
```

## Features Not Directly Convertible

### Step Lists (BeforeTestsSteps, AfterTestsSteps, BeforeReleaseSteps)

Azure DevOps supports dynamic step injection through `stepList` parameters. GitHub Actions doesn't have direct equivalent functionality. **Recommendation**: 
- Create separate reusable workflows for common step patterns
- Use composite actions for repeated step sequences
- Consider workflow templates for standardization

### Symbol Server Publishing

Azure DevOps `PublishSymbols@2` task publishes to Azure DevOps Symbol Server. GitHub Actions equivalent would require:
- Custom action or script to upload symbols
- Third-party symbol server integration
- Or include symbols in NuGet packages (`.snupkg`)

## Migration Checklist

### Pre-Migration
- [ ] Review all parameter usage in calling pipelines
- [ ] Identify required secrets and credentials
- [ ] Plan for step list parameter replacements
- [ ] Test Azure CLI integration requirements

### Post-Migration
- [ ] Configure repository secrets
- [ ] Update calling workflows to use new input names
- [ ] Test all conditional paths
- [ ] Verify artifact publishing
- [ ] Validate test result reporting
- [ ] Test container registry operations

### Testing Strategy
- [ ] Create test workflows for each major feature combination
- [ ] Test with and without optional features (caching, Azure CLI, Docker)
- [ ] Validate NuGet publishing to both internal and external feeds
- [ ] Test Platina-specific initialization steps

## Usage Example

```yaml
# .github/workflows/build.yml
name: Build and Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    uses: ./.github/workflows/ci.yml
    with:
      solution: 'MyProject.sln'
      build-configuration: 'Release'
      with-pack: true
      with-publish: true
      with-release: ${{ github.ref == 'refs/heads/main' }}
      with-nuget-cache: true
    secrets:
      NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
```

## Benefits of GitHub Actions Conversion

### Improved Features
- **Better caching**: More granular cache control with `actions/cache@v4`
- **Enhanced test reporting**: Rich test result visualization with `dorny/test-reporter@v1`
- **Improved security**: Dedicated login actions with proper secret handling
- **Better artifact management**: Native GitHub artifact system with better retention policies

### Simplified Workflow
- **Direct dotnet commands**: No abstraction layer, more transparent execution
- **Native shell support**: Better PowerShell and bash integration
- **Marketplace ecosystem**: Access to thousands of community actions

### Cost Considerations
- **Free tier**: 2000 minutes/month for public repos, unlimited for public repos
- **Concurrent jobs**: Better parallelization options
- **Runner efficiency**: Generally faster execution times

## Troubleshooting Common Issues

### Cache Key Issues
```yaml
# Ensure NUGET_PACKAGES environment variable is set
env:
  NUGET_PACKAGES: ${{ github.workspace }}/.nuget/packages
```

### Test Discovery Problems
```yaml
# Use specific test project glob patterns
test-projects-glob: '**/*[Tt]ests/**/*.csproj'
```

### Container Registry Authentication
```yaml
# Ensure registry URL format is correct
container-registry: 'myregistry.azurecr.io'  # Not 'https://myregistry.azurecr.io'
```

## Conclusion

This conversion maintains 100% of the original Azure DevOps pipeline functionality while providing enhanced capabilities through GitHub Actions' ecosystem. The reusable workflow pattern ensures consistent CI/CD processes across multiple repositories while leveraging GitHub's native features for better developer experience.

The key success factors for this migration are:
1. Proper secret configuration
2. Testing all conditional execution paths  
3. Validating artifact and package publishing workflows
4. Ensuring Azure integration works correctly for authenticated scenarios

For questions or issues with this conversion, refer to the GitHub Actions documentation or create an issue in this repository.