# Build Docker Image Job

## Purpose

This template contains a job definition which downloads the given pipeline artifact and builds a container image from that artifact, publishing it to the supplied container registry.

## Usage

This job template assumes that the pipeline has generated a build artifact containing the files to be included in the container image. The `Dockerfile` is NOT in this build artifact: it is pulled from the repository where the pipeline is defined. The `job` uses the default `checkout: self`, so all files within the repository are available if needed in `prebuildSteps`

### Includes

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/azure-devops-templates

stages:
  # Build Stages here

  - stage: docker_publish
    displayName: Create Docker Image
    jobs:
      - template: jobs/build-docker-image/v1.yml@templates
        parameters:
          dockerImageName: container-name
          containerRegistryName: myRegistryServiceConnection
          artifactName: artifactCreatedInPreviousStage
          artifactZipName: zipFileInArtifact
          dockerFilePath: /path/to/Dockerfile
```

### Parameters

| Name                    | Type     | Description                                                         | Default Value                                 |
| ----------------------- | -------- | ------------------------------------------------------------------- | --------------------------------------------- |
| `dockerImageName`       | string   | The container image (repository) name.                              |                                               |
| `containerRegistryName` | string   | The name of the container registry service connection.              |                                               |
| `containerRegistryUrl`  | string   | An API key tied to the Atlassian username                           |                                               |
| `artifactName`          | object   | The name of the pipeline artifact to download.                      |                                               |
| `artifactZipName`       | string   | The zip file name within the artifact.                              |                                               |
| `dockerFilePath`        | string   | The path (within the repository) of the DockerFile                  | `$(Build.SourcesDirectory)/Dockerfile.simple` |
| `dockerBuildBasePath`   | string   | The working directory to use when staging the container build.      | `$(Pipeline.Workspace)/dockerFiles`           |
| `artifactTargetPath`    | string   | The path within the artifact where the container files are located  | ""                                            |
| `prebuildSteps`         | stepList | Additional steps to be executed prior to the container image build. | `[]`                                          |
