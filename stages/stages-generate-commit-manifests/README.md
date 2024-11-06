# Generate Kubernetes Manifests and Commit

## Purpose

This stage template is designed to generate Kubernetes manifests from [Helmfile][1] projects and then commit them to a repository which acts as a GitOps repository.

## Usage

### Use Case

```yaml
resources:
  repositories:
    # This is the GitOps repository
    - repository: ops_nonprod_cluster
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/ops-nonprod-cluster
    - repository: templates
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/azure-devops-templates

trigger: none
pr: none

stages:
  - template: stages/stages-generate-commit-manifests/v1.yml@templates
    parameters:
      environmentName: "test"
      namespace: "test"
      helmfileDirectory: ./
      argoRepoName: ops_nonprod_cluster
      argoBaseDir: apps/home-automation
```

## Parameters

| Name                | Type     | Description                                                      | Default Value |
| ------------------- | -------- | ---------------------------------------------------------------- | ------------- |
| `environmentName`   | string   | Name of the Azure DevOps environment for deployment.             |               |
| `helmfileDirectory` | string   | Directory where the `helmfile.yaml` file is located.             | `./`          |
| `argoRepoName`      | string   | Azure DevOps Service Connection name for your GitOps repository. |               |
| `argoBaseDir`       | string   | Base Directory to copy the generated templates to.               |               |
| `pregenerateSteps`  | stepList | Steps to be execute prior to manifest generation.                | `[]`          |

## Build Order

1. Execute the `deployment` step to the given `environmentName`. This forces any security or approvals in Azure DevOps.
2. Execute [Helmfile Template](/step_collections/helmfile-template/v1.yml).
3. Execute [Commit to GitOps Repo](/step_collections/commit-to-gitops-repo/v1.yml).

## Helmfile Environment and Namespace

Currently, the `environmentName` used for Helmfile is the same as the Azure DevOps environment. Additionally, to support multiple namespaces, the Helmfile release must define a namespace. See [Helmfile Template](/step_collections/helmfile-template/README.md) for more information.

## Version History

### 1.0.0 \[v1.yml\]

- Initial Creation

[1]: https://helmfile.readthedocs.io/en/latest/ "Helmfile Documenation"
