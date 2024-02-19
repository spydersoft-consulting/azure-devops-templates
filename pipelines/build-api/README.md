# Build API Project

## Purpose

This pipeline is designed to build a repository containing a .Net API.

## Usage

### Use Case

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/azure-devops-templates
    ## Reference your Helmfile repository.
    - repository: helmfileconfig
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/techradar-helm-config

trigger: none
pr: none

extends:
  template: pipelines/build-api/v1.yml@templates
  parameters:
    # Parameters here
```

## Parameters

### API Build Parameters

| Name                 | Type   | Description                                                           | Default Value |
| -------------------- | ------ | --------------------------------------------------------------------- | ------------- |
| `publishProject`     | string | Projects to Publish. See [Specifying Projects](#specifying-projects). |               |
| `buildProject`       | string | Projects to Build. See [Specifying Projects](#specifying-projects).   |               |
| `netCoreVersion`     | string | NET SDK Version. See [UseDotNet@2][1] documentation for syntax.       | `6.0.x`       |
| `buildConfiguration` | string | Build Configuration                                                   | `Release`     |

### API Test Parameters

| Name            | Type    | Description                                                        | Default Value        |
| --------------- | ------- | ------------------------------------------------------------------ | -------------------- |
| `execute_tests` | boolean | If `true`, `dotnet test` will execute.                             | `false`              |
| `test_projects` | string  | Projects to Test. See [Specifying Projects](#specifying-projects). | `**/*tests/*.csproj` |

### API Sonarqube Parameters

| Name                       | Type    | Description                                                                                              | Default Value |
| -------------------------- | ------- | -------------------------------------------------------------------------------------------------------- | ------------- |
| `execute_sonar`            | boolean | `true` will execute Sonarqube analysis on the BFF API project.                                           | `false`       |
| `sonar_endpoint_name`      | string  | Name of the Azure DevOps Service Connection for SonarQube/SonarCloud. Required when `executeSonar=true`. |               |
| `sonar_project_key`        | string  | SonarQube/SonarCloud Project Key. Required when `executeSonar=true`.                                     |               |
| `sonar_project_name`       | string  | SonarQube/SonarCloud Project Name. Required when `executeSonar=true`.                                    |               |
| `use_sonarcloud`           | boolean | `true` to use SonarCloud, `false` to use SonarQube.                                                      | `false`       |
| `sonar_cloud_organization` | string  | Name of the SonarCloud Organization. Required when `useSonarcloud=true`.                                 | ""            |
| `sonar_extra_properties`   | string  | Extra properties to pass to Sonar analysis.                                                              | ""            |

### Build Artifact Parameters

| Name           | Type   | Description                                      | Default Value |
| -------------- | ------ | ------------------------------------------------ | ------------- |
| `artifactName` | string | The name to use when publishing to the pipeline. |               |

### Docker Container Parameters

| Name                         | Type    | Description                                                   | Default Value                                 |
| ---------------------------- | ------- | ------------------------------------------------------------- | --------------------------------------------- |
| `buildAndPublishDockerImage` | boolean | `true` to build and publish the docker image.                 | `false`                                       |
| `artifactZipName`            | string  | The name of the artifact to download for the docker build.    |                                               |
| `dockerImageName`            | string  | The name of the container image                               |                                               |
| `containerRegistryName`      | boolean | The Azure Service Connection name for the container registry. | `proget_docker`                               |
| `dockerFilePath`             | string  | Path to the Dockerfile                                        | `$(Build.SourcesDirectory)/Dockerfile.simple` |

### Helmfile Config Parameters

| Name                   | Type                                                      | Description                                                                                                                                                                                 | Default Value |
| ---------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| `updateHelmConfig`     | boolean                                                   | When `true`, update the `helmfileRepoName` repository with updated tags, based on the `imageTagVariableName`. Executes [this step collection](/step_collections/process-cicd-script/v1.yml) | `false`       |
| `helmfileRepoName`     | string                                                    | The name of the repository pipeline resource referencing the Helmfile Config repository for this project.                                                                                   |               |
| `imageTagVariableName` | The variable name to use when executing the CI/CD script. | string                                                                                                                                                                                      |               |

### Hooks

| Name            | Type     | Description                           | Default Value |
| --------------- | -------- | ------------------------------------- | ------------- |
| `prebuildSteps` | stepList | Steps executed prior to the API Build | `[]`          |
| `preTestSteps`  | stepList | Steps executed prior to the API Test  | `[]`          |

## Build Order

1. Checkout repository with full history (for Sonar Analysis).
2. Set .NET SDK Version.
3. Setup and Execute GitVersion.
4. Execute [DotNet API Build](/step_collections/dotnet-api-build-steps/v1.yml) with given parameters.
5. Publish the entirety of `$(Build.ArtifactStagingDirectory)` with `artifactName` to the pipeline.
6. [Build and Publish Docker image](/jobs/build-docker-image/v1.yml), if selected
7. [Update Helm Configuration](/step_collections/process-cicd-script/v1.yml), if selected.

## Specifying Projects

For the parameters which specify a list of projects, the [DotNetCoreCLI@2][2] task supports file matching patterns, including wildcards and exclusion patterns. See [file matching patterns reference][3] for more detail.

## SonarQube / SonarCloud Analysis

This pipeline is configured to allow for execution of SonarQube/SonarCloud analysis using the [SonarQube Extension for Azure Pipelines][4]. The pipeline supports both SonarQube (OnPremise) and SonarCloud (SaaS) instances, however the configuration differs slightly. While both require a Service Connection to be configured in the project, SonarCloud also requires you to supply an organization name. |

## Version History

### 1.0.0 \[v1.yml\]

- Initial Creation

[1]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/use-dotnet-v2?view=azure-pipelines "UseDotNet@2 Documentation"
[2]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/dotnet-core-cli-v2?view=azure-pipelines "DotNetCoreCLI@2 Documentation"
[3]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/file-matching-patterns?view=azure-devops "File matching patterns reference"
[4]: https://marketplace.visualstudio.com/items?itemName=SonarSource.sonarqube "SonarQube Extension for Azure Pipelines"
