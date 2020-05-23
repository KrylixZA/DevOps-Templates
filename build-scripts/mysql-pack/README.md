# MySQL pack
This directory contains the necessary steps to setup automatic packing of MySQL scripts into a deployable nuget package.

# Steps
1. Create a reposity and initialize it with a `azure-pipelines.yml` file at the root-level of your repository.
2. Browse to the root of your build pipeline and select `Edit`.
3. Select `Variables`.
4. Add the following three variables:
    1. `MajorVersion` = 1
    2. `MinorVersion` = 0
    3. `Patchversion` = 0
5. Add/modify your `azure-pipelines.yml` with the following:
    ``` YAML
    name: $(MajorVersion).$(MinorVersion).$(PatchVersion).$(Rev:r)

    trigger:
    branches:
      include:
      - '*'
    paths:
      include:
      - '*'

    pool:
      vmImage: 'windows-2019'

    variables:
      BuildConfiguration: 'Release'

    jobs:
    - job: 'Build'
      steps:
      - checkout: self
          fetchDepth: 1
          clean: true

      - task: PowerShell@2
          displayName: 'Set build number'
          inputs:
          targetType: 'inline'
          script: |
              $branchName = "$(Build.SourceBranchName)";
              $buildVersion = "$(Build.BuildNumber)";
              $nugetPkgVersion = $buildVersion;
              If (-not([string]::IsNullOrWhiteSpace($branchName)) -and ($branchName -ne "master")) {
                  $nugetPkgVersion = "$nugetPkgVersion-$branchName";
              }
              Write-Host "Package version: $nugetPkgVersion";
              Write-Host "##vso[build.updateBuildNumber]$nugetPkgVersion";

      - task: NuGetCommand@2
          displayName: 'nuget pack'
          inputs:
            command: 'pack'
            packagesToPack: '**/*.nuspec'
            configuration: '$(BuildConfiguration)'
            versioningScheme: 'byEnvVar'
            versionEnvVar: 'Build.BuildNumber'
            buildProperties: 'version="$(Build.BuildNumber)"'

      - task: DotNetCoreCLI@2
          displayName: 'dotnet nuget push'
          inputs:
            command: 'push'
            packagesToPush: '$(Build.ArtifactStagingDirectory)/*.nupkg'
            nuGetFeedType: 'internal'
            publishVstsFeed: '404449e0-6d24-4a4e-bc3e-4634d3f54a5a'
    ```