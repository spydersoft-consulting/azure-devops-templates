# .NET 6+ Library Build Pipeline Template

## Purpose

This template is designed to perform activities on a .Net 6+ Library. It is an `extends` template type, meaning that the logic is controlled within the template, and other files only define parameters and extend the core functionality.

## Usage

### Simple Use Case

```yaml
trigger: none
pr: none

resources:
  repositories:
    - repository: templates
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/azure-devops-templates

extends:
  template: src/pipelines/extends/dotnet-library-build/v1.yml@azure-pipelines
  parameters:
    buildProject: source/Spydersoft.Core.Hosting/Spydersoft.Core.Hosting.csproj
```

## Parameters

| Name                     | Type     | Description                                                                                              | Default Value                                   |
| ------------------------ | -------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| `buildConfiguration`     | string   | Configuration name for build                                                                             | `Release`                                       |
| `dotnetSdk`              | string   | NET SDK Version. See [UseDotNet@2][1] documentation for syntax.                                          | `5.0.x`                                         |
| `projectsToBuild`        | string   | Projects to Build. See [Specifying Projects](#specifying-projects).                                      | `**/*.csproj`                                   |
| `projectsToPack`         | string   | Projects to Pack. See [Specifying Projects](#specifying-projects).                                       | `**/*.csproj`                                   |
| `buildArtifactName`      | string   | The name for the artifact package in the Azure DevOps pipeline.                                          | `packages`                                      |
| `projectsToRestore`      | string   | Projects to Restore. See [Specifying Projects](#specifying-projects).                                    | `**/*.csproj`                                   |
| `useNugetConfigFile`     | boolean  | `true` will specify `config` and use the `nugetConfigPath`. `false` will specify `select` mode.          |
| `nugetConfigPath`        | string   | Path to the nuget.config file in the code repository. If empty, only nuget.org is used.                  | `$(Build.SourcesDirectory)/source/nuget.config` |
| `vstsFeedId`             | string   | The Azure DevOps Artifact feed ID for publishing.                                                        | `0aa0c07a-38e6-4fad-ab6f-7aa010d18aeb`          |
| `executeTests`           | boolean  | Whether or not to execute `dotnet test` for this pipeline.                                               | `false`                                         |
| `projectsToTest`         | string   | Projects for Test Execution. See [Specifying Projects](#specifying-projects).                            | `**/*.UnitTests/*.csproj`                       |
| `executeSonar`           | boolean  | Whether or not to execute SonarQube/Cloud analysis for this pipeline.                                    | `false`                                         |
| `useSonarcloud`          | boolean  | `true` to use SonarCloud, `false` to use SonarQube.                                                      | `false`                                         |
| `sonarCloudOrganization` | string   | Name of the SonarCloud Organization. Required when `useSonarcloud=true`.                                 |                                                 |
| `sonarEndpointName`      | string   | Name of the Azure DevOps Service Connection for SonarQube/SonarCloud. Required when `executeSonar=true`. |                                                 |
| `sonarProjectKey`        | string   | SonarQube/SonarCloud Project Key. Required when `executeSonar=true`.                                     |
| `sonarProjectName`       | string   | SonarQube/SonarCloud Project Name. Required when `executeSonar=true`.                                    |                                                 |
| `preBuildSteps`          | stepList | Steps executed immediately before `dotnet restore`. See [Build Order](#build-order).                     | `[]`                                            |
| `preTestSteps`           | stepList | Steps executed immediately before `dotnet restore`. See [Build Order](#build-order).                     | `[]`                                            |

### Build Order

1. Checkout repository with full history (for Sonar Analysis).
2. Set .NET SDK Version.
3. Setup and Execute GitVersion.
4. Setup SonarQube/SonarCloud analysis if `executeSonar` is `true`.
5. Execute `preBuildSteps`.
6. Execute Nuget Authentication for pipeline.
7. Execute `dotnet restore` using associated parameters.
8. Execute `dotnet build` using associated parameters.
9. If `executeTests`
   1. Execute `preTestSteps`.
   2. Execute `dotnet test` using associated parameters.
   3. Copy test results files to a temporary directory.
   4. Publish Test Results to pipeline results.
   5. Publish Code Coverate results to pipeline results.
10. Execute SonarQube/SonarCloud analysis if `executeSonar` is `true`.
11. Execute `dotnet pack` using associated parameters.
12. Publish the contents of `$(Build.ArtifactStagingDirectory)` to the pipeline, using the `buildArtifactName`.
13. Push any `*.nupkg` packages in `$(Build.ArtifactStagingDirectory)` to the artifact feed represented by `vstsFeedId`.

### Specifying Projects

For the parameters which specify a list of projects, the [DotNetCoreCLI@2][2] task supports file matching patterns, including wildcards and exclusion patterns. See [file matching patterns reference][3] for more detail.

### SonarQube / SonarCloud Analysis

This pipeline is configured to allow for execution of SonarQube/SonarCloud analysis using the [SonarQube Extension for Azure Pipelines][4]. The pipeline supports both SonarQube (OnPremise) and SonarCloud (SaaS) instances, however the configuration differs slightly. While both require a Service Connection to be configured in the project, SonarCloud also requires you to supply an organization name.

## Version History

### 1.0.0 \[v1.yml\]

- Initial Creation

[1]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/use-dotnet-v2?view=azure-pipelines "UseDotNet@2 Documentation"
[2]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/dotnet-core-cli-v2?view=azure-pipelines "DotNetCoreCLI@2 Documentation"
[3]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/file-matching-patterns?view=azure-devops "File matching patterns reference"
[4]: https://marketplace.visualstudio.com/items?itemName=SonarSource.sonarqube "SonarQube Extension for Azure Pipelines"
