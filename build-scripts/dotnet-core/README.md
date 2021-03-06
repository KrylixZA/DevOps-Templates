# .NET Core
This directory contains the necessary steps to setup automatic build & publishing of your build artifacts, including coverage reports, etc.

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

      - task: SonarCloudPrepare@1
        displayName: 'Prepare for analysis'
        inputs:
          SonarCloud: 'SonarCloud'
          organization: 'headleysj'
          scannerMode: 'MSBuild'
          projectKey: 'Your-Project-Key'
          extraProperties: 'sonar.cs.opencover.reportsPaths=$(Common.TestResultsDirectory)/coverage.opencover.xml'

      - task: DotNetCoreCLI@2
        displayName: 'dotnet restore'
        inputs:
          command: 'restore'
          projects: './src'
          feedsToUse: 'select'
          vstsFeed: '404449e0-6d24-4a4e-bc3e-4634d3f54a5a'

      - task: DotNetCoreCLI@2
        displayName: 'dotnet build'
        inputs:
          command: 'build'
          projects: './src'
          arguments: '--no-restore --configuration $(BuildConfiguration) -p:Version=$(Build.BuildNumber)'

      - task: NuGetCommand@2
        displayName: 'nuget pack'
        inputs:
          command: 'pack'
          packagesToPack: '**/*.nuspec'
          configuration: '$(buildConfiguration)'
          versioningScheme: 'byEnvVar'
          versionEnvVar: 'Build.BuildNumber'
          buildProperties: 'version="$(Build.BuildNumber)"'

      - task: DotNetCoreCLI@2
        displayName: 'dotnet test'
        inputs:
          command: 'test'
          projects: './src'
          arguments: '--no-restore --configuration $(BuildConfiguration) /p:CollectCoverage=true /p:CoverletOutputFormat="cobertura%2cjson%2copencover" /p:CoverletOutput="$(Common.TestResultsDirectory)/" /p:MergeWith="$(Common.TestResultsDirectory)/coverage.json"'

      - task: DotNetCoreCLI@2
        displayName: 'dotnet nuget push'
        inputs:
          command: 'push'
          packagesToPush: '$(Build.ArtifactStagingDirectory)\*.nupkg'
          nuGetFeedType: 'internal'
          publishVstsFeed: '404449e0-6d24-4a4e-bc3e-4634d3f54a5a'

      - task: PublishBuildArtifacts@1
        displayName: 'Publish build artifacts'
        inputs:
          PathtoPublish: '$(Build.ArtifactStagingDirectory)'
          ArtifactName: 'drop'
          publishLocation: 'Container'

      - task: PublishCodeCoverageResults@1
        displayName: 'Publish code coverage results'
        inputs:
          codeCoverageTool: Cobertura
          summaryFileLocation: '$(Common.TestResultsDirectory)/coverage.cobertura.xml'

      - task: PublishTestResults@2
        displayName: 'Publish test results'
        inputs:
          testResultsFormat: 'VSTest'
          testResultsFiles: '**/**.trx'
          searchFolder: '$(Agent.TempDirectory)'
          mergeTestResults: true
          failTaskOnFailedTests: true
          buildConfiguration: '$(BuildConfiguration)'

      - task: SonarCloudAnalyze@1
        displayName: 'SonarCloud analysis'

      - task: SonarCloudPublish@1
        displayName: 'SonarCloud publish'
        inputs:
          pollingTimeoutSec: '300'
    ```