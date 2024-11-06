# Build Backend for Frontend

## Purpose

This pipeline is designed to build a repository containing a SPA application (currently using Yarn) and a .Net API which serves as the backend.

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
  template: pipelines/build-bff/v1.yml@templates
  parameters:
    # Parameters here
```

## Parameters

### BFF API Build Parameters

| Name                 | Type   | Description                                                           | Default Value |
| -------------------- | ------ | --------------------------------------------------------------------- | ------------- |
| `publishProject`     | string | Projects to Publish. See [Specifying Projects](#specifying-projects). |               |
| `buildProject`       | string | Projects to Build. See [Specifying Projects](#specifying-projects).   |               |
| `netCoreVersion`     | string | NET SDK Version. See [UseDotNet@2][1] documentation for syntax.       | `6.0.x`       |
| `buildConfiguration` | string | Build Configuration                                                   | `Release`     |

### BFF API Test Parameters

| Name              | Type    | Description                                                        | Default Value        |
| ----------------- | ------- | ------------------------------------------------------------------ | -------------------- |
| `bffExecuteTests` | boolean | If `true`, `dotnet test` will execute.                             | `false`              |
| `bffTestProjects` | string  | Projects to Test. See [Specifying Projects](#specifying-projects). | `**/*tests/*.csproj` |

### BFF API Sonarqube Parameters

| Name                        | Type    | Description                                                                                              | Default Value |
| --------------------------- | ------- | -------------------------------------------------------------------------------------------------------- | ------------- |
| `bffExecuteSonar`           | boolean | `true` will execute Sonarqube analysis on the BFF API project.                                           | `false`       |
| `bffSonarEndpointName`      | string  | Name of the Azure DevOps Service Connection for SonarQube/SonarCloud. Required when `executeSonar=true`. |               |
| `bffSonarProjectKey`        | string  | SonarQube/SonarCloud Project Key. Required when `executeSonar=true`.                                     |               |
| `bffSonarProjectName`       | string  | SonarQube/SonarCloud Project Name. Required when `executeSonar=true`.                                    |               |
| `bffUseSonarCloud`          | boolean | `true` to use SonarCloud, `false` to use SonarQube.                                                      | `false`       |
| `bffSonarCloudOrganization` | string  | Name of the SonarCloud Organization. Required when `useSonarCloud=true`.                                 | ""            |
| `bffSonarExtraProperties`   | string  | Extra properties to pass to Sonar analysis.                                                              | ""            |

### UI Build Parameters

| Name                 | Type    | Description                                               | Default Value                            |
| -------------------- | ------- | --------------------------------------------------------- | ---------------------------------------- |
| `uiSourceDirectory`  | boolean | Source directory for the SPA.                             | `src/ui`                                 |
| `uiCodeCoverageFile` | string  | Location of the code coverage file generated after tests. | `output/coverage/cobertura-coverage.xml` |
| `uiUnitTestFile`     | string  | Location of the Unit Test Results file.                   | `output/test/junit.xml`                  |

### UI Sonarqube Parameters

| Name                       | Type    | Description                                                                                              | Default Value              |
| -------------------------- | ------- | -------------------------------------------------------------------------------------------------------- | -------------------------- |
| `uiExecuteSonar`           | boolean | `true` will execute Sonarqube analysis on the BFF API project.                                           | `false`                    |
| `uiSonarEndpointName`      | string  | Name of the Azure DevOps Service Connection for SonarQube/SonarCloud. Required when `executeSonar=true`. |                            |
| `uiUseSonarCloud`          | boolean | `true` to use SonarCloud, `false` to use SonarQube.                                                      | `false`                    |
| `uiSonarCloudOrganization` | string  | Name of the SonarCloud Organization. Required when `useSonarCloud=true`.                                 | ""                         |
| `uiSonarConfigFile`        | string  | Path to the sonar project properties file for the UI.                                                    | `sonar-project.properties` |

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
4. Execute [Yarn Build and Test](/step_collections/yarn-build-test/v1.yml) with given parameters.
5. Execute [DotNet API Build](/step_collections/dotnet-api-build-steps/v1.yml) with given parameters.
6. Publish the entirety of `$(Build.ArtifactStagingDirectory)` with `artifactName` to the pipeline.
7. [Build and Publish Docker image](/jobs/build-docker-image/v1.yml), if selected
8. [Update Helm Configuration](/step_collections/process-cicd-script/v1.yml), if selected.

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
