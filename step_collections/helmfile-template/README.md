# Generate Kubernetes Manifests with Helmfile Template

## Purpose

This step collection is designed to generate Kubernetes manifests from [Helmfile][1] projects.

## Usage

### Use Case

```yaml
resources:
  repositories:
    # This is the GitOps repository
    - repository: templates
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/azure-devops-templates

trigger: none
pr: none

stages:
  - stage: generate_commit_manifest
    displayName: Deploy To my Environment
    jobs:
      - job: generate_manifests
        pool:
          name: Default
        steps:
          - template: ../../step_collections/helmfile-template/v1.yml
            parameters:
              # parameters
```

## Parameters

| Name                | Type     | Description                                                                                | Default Value |
| ------------------- | -------- | ------------------------------------------------------------------------------------------ | ------------- |
| `environmentName`   | string   | Name of the Azure DevOps environment for deployment.                                       |               |
| `helmfileDirectory` | string   | Directory where the `helmfile.yaml` file is located.                                       | `./`          |
| `displayPrefix`     | string   | A prefix for the steps in the template. Useful when executed multiple times in a pipeline. | `App`         |
| `pregenerateSteps`  | stepList | Steps to be execute prior to manifest generation.                                          | `[]`          |

## Build Order

1. Login to the given Kubernetes cluster.
2. Execute `pregenerateSteps`.
3. Download [Helmfile][1].
4. Execute `helmfile repos` with the given `environmentName` to ensure all repositories are configured.
5. Execute `helmfile template` with the following parameters:
   1. `environment` - The supplied `environmentName`
   2. `output-dir-template` - `$(Build.ArtifactStagingDirectory)/output/{{ .Release.Namespace }}`
6. Publish the generated files to a pipeline artifiact with a name equal to the `environmentName` parameter.

## Helmfile Environment and Namespace

Currently, the `environmentName` used for Helmfile is the same as the Azure DevOps environment. Additionally, to support multiple namespaces, the Helmfile release must define a namespace.

The `output-dir-template` places the generated output in `$(Build.ArtifactStagingDirectory)/output`, followed by a folder for the namespace. The `output` folder is then zipped and published to the pipeline for future use.

## Version History

### 1.0.0 \[v1.yml\]

- Initial Creation

[1]: https://helmfile.readthedocs.io/en/latest/ "Helmfile Documenation"
