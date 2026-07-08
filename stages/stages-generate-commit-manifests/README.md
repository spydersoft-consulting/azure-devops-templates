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

| Name                | Type     | Description                                                                                                                                                                                                                                      | Default Value |
| ------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------- |
| `environmentName`   | string   | Name of the Azure DevOps environment for deployment.                                                                                                                                                                                             |               |
| `helmfileDirectory` | string   | Directory where the `helmfile.yaml` file is located.                                                                                                                                                                                             | `./`          |
| `argoRepoName`      | string   | Azure DevOps Service Connection name for your GitOps repository.                                                                                                                                                                                 |               |
| `argoBaseDir`       | string   | Base Directory to copy the generated templates to.                                                                                                                                                                                               |               |
| `pregenerateSteps`  | stepList | Steps to be execute prior to manifest generation.                                                                                                                                                                                                | `[]`          |
| `dependsOn`         | object   | Stage(s) this one waits on. Leave empty (default) to keep Azure Pipelines' implicit "depends on the previous stage in the file" behavior; pass explicitly to wire multiple calls of this template into one pipeline with cross-stage conditions. | `[]`          |
| `condition`         | string   | Stage-level condition. Combine with `dependsOn` to skip this stage (and therefore its environment's approval check) entirely under some condition, e.g. a beta build that should never reach a release-only stage.                               | `succeeded()` |

## Build Order

1. Execute the `deployment` step to the given `environmentName`. If that environment has approval checks configured in Azure DevOps, this pauses here until approved.
2. Execute [Helmfile Template](/step_collections/helmfile-template/v1.yml) (runs in parallel with step 1 — manifest generation itself isn't gated).
3. Execute [Commit to GitOps Repo](/step_collections/commit-to-gitops-repo/v1.yml) — waits on **both** steps 1 and 2. This is the step that actually triggers Argo to sync, so it's the one an environment's approval check needs to block; it depending only on manifest generation (not the deployment step) was a real bug prior to this version, where an approval check on `environmentName` never actually gated anything.

## Helmfile Environment and Namespace

Currently, the `environmentName` used for Helmfile is the same as the Azure DevOps environment. Additionally, to support multiple namespaces, the Helmfile release must define a namespace. See [Helmfile Template](/step_collections/helmfile-template/README.md) for more information.

## Version History

### 1.0.0 \[v1.yml\]

- Initial Creation

[1]: https://helmfile.readthedocs.io/en/latest/ "Helmfile Documenation"
