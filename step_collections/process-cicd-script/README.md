# Process CICD Script

> [!WARNING]
> This collection of steps is EXTREMELY specific to my current infrastructure and has not yet been generalized. I would suggest using it as a template for your own work.

## Purpose

This step collection executes `scripts/process-cicd.ps1` in the supplied repository.

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
      - job: updateHelmConfigJob
        workspace:
          clean: all
        steps:
          - template: ../../step_collections/process-cicd-script/v1.yml
            parameters:
              repositoryResource: ${{ parameters.helmfileRepoName }}
              branchName: $(Build.SourceBranch)
              tagsCollection: ${{ parameters.imageTagVariableName }}=$(build.buildnumber)
              commitMessage: Updated ${{ parameters.imageTagVariableName }} to $(build.buildnumber)
```

## Parameters

| Name                 | Type   | Description                                                                              | Default Value     |
| -------------------- | ------ | ---------------------------------------------------------------------------------------- | ----------------- |
| `repositoryResource` | string | Name of the pipeline resource representing the repository on which to execute the steps. |                   |
| `branchName`         | string | Name of the Git branch to use.                                                           | `refs/heads/main` |
| `tagsCollection`     | string | Tags to update. See [Updating Tags](#updating-tags) for details.                         |                   |
| `commitMessage`      | string | Commit message to send to the script.                                                    |                   |

## Build Order

1. Checkout the supplied `repositoryResource`
2. Configure Python.
3. Configure Git.
4. Execute `scripts/process-cicd.ps1` in the `repositoryResource` with the following parameters:
   1. `branchName` - supplied `branchName` parameter.
   2. `tags` - supplied `tagsCollection`.
   3. `message` - supplied `commitMessage`.

## Updating Tags

Generally, the `process-cicd.ps1` script is used to update tags for containers in a Helmfile repository when new builds are available. A good example of a Helmfile configuration repository as I have built it is the [Technology Radar][1] repository.

The tag collection is a list of comma-separated values formatted as `tagname=version`. So, to pass multiple tags, the format is `tagname=version,tagname2=version2`, and so on.

## Version History

### 1.0.0 \[v1.yml\]

- Initial Creation

[1]: https://github.com/spydersoft-consulting/techradar-helm-config "Technology Radar Helmfile Configuration Repository."
