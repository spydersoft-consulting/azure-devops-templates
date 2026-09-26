# Build Multi-Container

## Purpose

Builds a single solution that publishes **multiple** deployable projects (e.g. an API
and a background processor sharing one repo) from one build number, then optionally
builds/publishes a docker image per project, packages/publishes NuGet packages, packages
Helm charts, and updates a Helmfile config repository. One atomic build avoids
version-mismatch drift between artifacts and produces a single Sonar analysis per build,
instead of a separate pipeline per artifact racing/overwriting each other's scans.

## Usage

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/azure-devops-templates
    - repository: helmfileconfig
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/platform-helm-config

trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

extends:
  template: pipelines/build-multi-container/v1.yml@templates
  parameters:
    vmImage: ubuntu-latest
    buildProject: src/MySolution.slnx

    publishProjects:
      - name: my_api
        project: src/MyApi/MyApi.csproj
        artifactName: myApi

    executeTests: true
    testProjects: "src/**/*.UnitTests/*.csproj"

    executeSonar: true
    sonarProjectKey: my-org_my-repo
    sonarProjectName: MyRepo
    useSonarCloud: true
    sonarEndpointName: sonarcloud-my-org
    sonarCloudOrganization: my-org

    buildAndPublishDockerImages: true
    containerRegistryName: github-my-org-docker
    dockerImages:
      - name: my_api
        dockerImageName: my-org/my-api
        dockerFilePath: Dockerfile
        artifactName: myApi
        artifactZipName: MyApi

    helmfileRepoName: helmfileconfig
    helmTagsCollection: "my_api=$(build.buildnumber)"
```

### Parameters

#### GitVersion

| Name             | Type   | Description                              | Default Value |
| ---------------- | ------ | ---------------------------------------- | ------------- |
| `gitVersionSpec` | string | [GitVersion][1] tool version to install. | `6.0.x`       |

#### .NET Build

| Name                      | Type   | Description                                                                                         | Default Value                            |
| ------------------------- | ------ | --------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `buildProject`            | string | The solution (or csproj glob) to build.                                                             | `**/*.slnx`                              |
| `netCoreVersion`          | string | .NET SDK version. See [UseDotNet@2][2].                                                             | `8.0.x`                                  |
| `buildConfiguration`      | string | Build configuration.                                                                                | `Release`                                |
| `externalFeedCredentials` | string | NuGet service connection name used for `dotnet restore`.                                            | `SpydersoftGithub`                       |
| `nugetConfigPath`         | string | Path to `nuget.config`. Override for repos that keep it somewhere other than the checkout root.     | `$(Build.SourcesDirectory)/nuget.config` |
| `publishProjects`         | object | List of projects to publish as deployable artifacts. Each entry: `{ name, project, artifactName }`. | `[]`                                     |

#### NuGet Pack / Publish (optional)

Set `publishToNuget: true` to pack and push NuGet packages from the same build job. The
pack step runs `--no-build`, reusing the solution build above.

| Name                    | Type    | Description                                 | Default Value |
| ----------------------- | ------- | ------------------------------------------- | ------------- |
| `publishToNuget`        | boolean | If `true`, packs and pushes NuGet packages. | `false`       |
| `projectsToPack`        | string  | Projects/glob to pack.                      | `""`          |
| `externalPublishUrl`    | string  | NuGet feed URL to push to.                  | `""`          |
| `externalPublishApiKey` | string  | API key for `externalPublishUrl`.           | `""`          |

#### .NET Test

| Name           | Type    | Description                                                                                                                                                     | Default Value        |
| -------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| `executeTests` | boolean | If `true`, runs tests after the build.                                                                                                                          | `false`              |
| `testProjects` | string  | Projects to test. See [file matching patterns reference][3].                                                                                                    | `**/*tests/*.csproj` |
| `testRunner`   | string  | `vstest` or `mtp` — see [dotnet-multi-publish-build-steps's Test Runner Modes](/step_collections/dotnet-multi-publish-build-steps/README.md#test-runner-modes). | `vstest`             |

#### .NET Sonar (one project covering all C# in the solution)

| Name                     | Type    | Description                                                                          | Default Value |
| ------------------------ | ------- | ------------------------------------------------------------------------------------ | ------------- |
| `executeSonar`           | boolean | If `true`, runs Sonar prepare/analyze/publish around the build.                      | `false`       |
| `sonarEndpointName`      | string  | Service connection name for SonarQube/SonarCloud. Required when `executeSonar=true`. | `""`          |
| `sonarProjectKey`        | string  | SonarQube/SonarCloud project key.                                                    | `""`          |
| `sonarProjectName`       | string  | SonarQube/SonarCloud project name.                                                   | `""`          |
| `useSonarCloud`          | boolean | `true` for SonarCloud, `false` for SonarQube (on-prem).                              | `false`       |
| `sonarCloudOrganization` | string  | SonarCloud organization name. Required when `useSonarCloud=true`.                    | `""`          |
| `sonarExtraProperties`   | string  | Extra properties appended to the Sonar `extraProperties` block.                      | `""`          |

#### UI Build (optional)

Set `uiEnabled: true` to include a yarn build/test/Sonar pass (via
[yarn-build-test](/step_collections/yarn-build-test/v1.yml)) alongside the .NET build.

| Name                 | Type    | Description                                          | Default Value                            |
| -------------------- | ------- | ---------------------------------------------------- | ---------------------------------------- |
| `uiEnabled`          | boolean | If `true`, runs the yarn build/test step collection. | `false`                                  |
| `uiSourceDirectory`  | string  | Working directory for the yarn build.                | `src/ui`                                 |
| `uiNodeVersion`      | string  | Node version to install.                             | `22.x`                                   |
| `uiUseYarn2`         | boolean | `true` if the project uses Yarn 2+ (Berry).          | `true`                                   |
| `uiCodeCoverageFile` | string  | Path to the generated coverage file.                 | `output/coverage/cobertura-coverage.xml` |
| `uiUnitTestFile`     | string  | Path to the generated JUnit test results file.       | `output/test/junit.xml`                  |

#### UI Sonar (one project for the JS/TS code)

| Name                       | Type    | Description                                       | Default Value              |
| -------------------------- | ------- | ------------------------------------------------- | -------------------------- |
| `uiExecuteSonar`           | boolean | If `true`, runs Sonar analysis on the UI project. | `false`                    |
| `uiSonarEndpointName`      | string  | Service connection name for SonarQube/SonarCloud. | `""`                       |
| `uiUseSonarCloud`          | boolean | `true` for SonarCloud, `false` for SonarQube.     | `false`                    |
| `uiSonarCloudOrganization` | string  | SonarCloud organization name.                     | `""`                       |
| `uiSonarConfigFile`        | string  | Path to `sonar-project.properties`.               | `sonar-project.properties` |

#### Docker

One entry in `dockerImages` per container image to build & push.

| Name                          | Type    | Description                                                                                                                                                                                                                           | Default Value   |
| ----------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| `buildAndPublishDockerImages` | boolean | If `true`, adds a `docker_publish` stage.                                                                                                                                                                                             | `false`         |
| `dockerImages`                | object  | Each entry: `{ name, dockerImageName, dockerFilePath, artifactName, artifactZipName }`. `name` must be alphanumeric + underscore (used as the job identifier); `artifactName`/`artifactZipName` must match a `publishProjects` entry. | `[]`            |
| `containerRegistryName`       | string  | Container registry service connection name.                                                                                                                                                                                           | `proget_docker` |

#### Helm Chart

One entry in `helmCharts` per chart to package & push as an OCI artifact.

| Name                        | Type    | Description                                                                                                                                  | Default Value                          |
| --------------------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| `buildAndPublishHelmCharts` | boolean | If `true`, adds a `chart_publish` stage.                                                                                                     | `false`                                |
| `helmCharts`                | object  | Each entry: `{ name, chartPath }` — `name` must match the chart's own `Chart.yaml` `name`; `chartPath` is the repo-relative chart directory. | `[]`                                   |
| `helmChartRegistry`         | string  | OCI registry to push charts to.                                                                                                              | `ghcr.io/spydersoft-consulting/charts` |

#### Helmfile Config

| Name                 | Type    | Description                                                                                                                                                        | Default Value |
| -------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------- |
| `updateHelmConfig`   | boolean | If `true`, adds an `updateHelmConfig` stage.                                                                                                                       | `false`       |
| `helmfileRepoName`   | string  | Pipeline resource name for the Helmfile config repository.                                                                                                         | `""`          |
| `helmTagsCollection` | string  | Comma-separated tag assignments, e.g. `"apiImageTag=$(build.buildnumber),chartVersion=$(build.buildnumber)"`. A single commit updates every listed tag atomically. | `""`          |

#### Stage Wiring

These let a consumer splice externally-defined stages (e.g. an integration-test stage
from another template) into the serial chain, by re-pointing what a built-in stage waits
on. Defaults reproduce the standalone order: `Build` -> `docker_publish` ->
`updateHelmConfig`. If `buildAndPublishHelmCharts` is also enabled, pass
`updateHelmConfigDependsOn: [docker_publish, chart_publish]` explicitly — those two run
in parallel off `Build`, independent of one another, so there's no single default that
covers both without opting in.

| Name                        | Type   | Description                           | Default Value      |
| --------------------------- | ------ | ------------------------------------- | ------------------ |
| `dockerPublishDependsOn`    | object | Stages `docker_publish` depends on.   | `[Build]`          |
| `chartPublishDependsOn`     | object | Stages `chart_publish` depends on.    | `[Build]`          |
| `updateHelmConfigDependsOn` | object | Stages `updateHelmConfig` depends on. | `[docker_publish]` |

#### Hooks

| Name            | Type     | Description                                                                          | Default Value |
| --------------- | -------- | ------------------------------------------------------------------------------------ | ------------- |
| `prebuildSteps` | stepList | Steps executed before `dotnet restore`.                                              | `[]`          |
| `preTestSteps`  | stepList | Steps executed immediately before the test step.                                     | `[]`          |
| `setupSteps`    | stepList | Steps executed before GitVersion setup (e.g. authenticating an additional resource). | `[]`          |

#### Pool Selection

| Name       | Type   | Description                                                                                              | Default Value |
| ---------- | ------ | -------------------------------------------------------------------------------------------------------- | ------------- |
| `poolName` | string | Self-hosted agent pool name. Ignored if `vmImage` is set.                                                | `Default`     |
| `vmImage`  | string | Microsoft-hosted agent image (e.g. `ubuntu-latest`). Takes precedence over `poolName` when both are set. | `""`          |

## Build Order

1. Checkout repository with full history (for Sonar analysis).
2. Run `setupSteps`, if any.
3. Setup and execute GitVersion.
4. Run the yarn build/test/Sonar step collection, if `uiEnabled`.
5. Execute [dotnet-multi-publish-build-steps](/step_collections/dotnet-multi-publish-build-steps/README.md) (restore, build, test, Sonar, publish each `publishProjects` entry).
6. Pack and push NuGet packages, if `publishToNuget`.
7. `docker_publish` stage: build & push each `dockerImages` entry, if `buildAndPublishDockerImages`.
8. `chart_publish` stage: package & push each `helmCharts` entry, if `buildAndPublishHelmCharts`.
9. `updateHelmConfig` stage: commit `helmTagsCollection` to the Helmfile config repo, if `updateHelmConfig`.

## Version History

### 1.0.0 \[v1.yml\]

- Initial Creation

### 1.1.0 \[v1.yml\]

- Added `testRunner` parameter, forwarded to `dotnet-multi-publish-build-steps`.

[1]: https://gitversion.net/docs/ "GitVersion documentation"
[2]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/use-dotnet-v2?view=azure-pipelines "UseDotNet@2 Documentation"
[3]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/file-matching-patterns?view=azure-devops "File matching patterns reference"
