# Azure DevOps Pipeline Templates

This repository contains a number of templates that can expedite and standardize build and deploy activities in Azure DevOps.

## Usage

Templates can be [extended][1] or [included][2].

### Extend a Template

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/azure-devops-templates

trigger: none
pr: none

extends:
  template: pipelines/build-api/v1.yml@templates
  parameters:
    # set template parameters here
```

### Include a Template

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/azure-devops-templates

trigger: none
pr: none

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        workspace:
          clean: all
        pool:
          name: Default
          demands: agent.os -equals Linux
        steps:
          - checkout: self
            fetchDepth: 0
            clean: true
          - templates: step_collections/yarn-build-test/v1.yml@templates
            parameters:
              # set template parameters here
```

## Template structure

Each template should reside in a folder that reflects the template's name, and contain the following:

- A `README.md` file documenting the template
- Template files, versioned as `v<version>.yml`

This structure allows a single documentation file, with a changelog, for each step template, and supports multiple versions of the templates.

## Repository Structure

The repository structure is as follows:

- [jobs](./jobs/) - Templates in here should be treated primarily as [include][2] templates. Each template should represent a single job.
- [pipelines](./pipelines/) - Templates in here should be treated as templates to be [extended][1], as opposed to [included][2]. They generally represent full, multi-stage pipelines.
- [stages](./stages/) - Templates in here should be treated primarily as [include][2] templates. Each template should represent a single stage with one or more jobs.
- [step_collections](./step_collections/) - Templates in here should be treated primarily as [include][2] templates. Each template should represent a set of steps, with no job or stage definitions.

## Automated Styling

This repository is setup to use `yarn`, `prettier`, `lint-staged`, and `husky` to ensure files are formatted prior to commit. Make sure you have the modern yarn executable installed, and run `yarn install` to ensure everything is configured.

[1]: https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops&pivots=templates-extends "Extend Templates"
[2]: https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops&pivots=templates-includes "Include Templates"
