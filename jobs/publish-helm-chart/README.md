# Publish Helm Chart Job

## Purpose

This template contains a job definition which packages a local Helm chart (from the repository where the pipeline is defined) and pushes it to an OCI registry.

## Usage

The chart is read directly from the repository checkout (`checkout: self`), not from a pipeline artifact -- unlike `jobs/build-docker-image/v1.yml`, there is no prior build/publish stage producing chart content, so this job checks out the source repository itself.

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

  - stage: chart_publish
    displayName: Publish Helm Chart
    jobs:
      - template: jobs/publish-helm-chart/v1.yml@templates
        parameters:
          chartName: myapp
          chartPath: charts/myapp
          chartVersion: $(build.buildnumber)
          containerRegistryName: myRegistryServiceConnection
```

### Parameters

| Name                    | Type   | Description                                                                                                                                                                    | Default Value                          |
| ----------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------- |
| `chartName`             | string | The chart name, matching `Chart.yaml`'s `name` -- used to locate the packaged `.tgz`.                                                                                          |                                        |
| `chartPath`             | string | The path (within the repository) to the chart directory.                                                                                                                       |                                        |
| `chartVersion`          | string | The version to stamp on the chart and app version at package time.                                                                                                             |                                        |
| `ociRegistry`           | string | The OCI registry + path to push the chart to.                                                                                                                                  | `ghcr.io/spydersoft-consulting/charts` |
| `containerRegistryName` | string | The name of the container registry service connection, reused to authenticate the OCI push (the same registry that hosts container images also hosts charts as OCI artifacts). |                                        |
