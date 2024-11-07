# Publish MD to Confluence Template

## Purpose

This template is designed to publish Markdown files from a repository into Confluence, and supports Mermaid diagrams by converting the Mermaid code into an image.

This pipeline uses the [Mermaid CLI][1] and [md-to-conf][2] to convert Mermaid code into images and to publish MD files to Confluence.

## Usage

### Use Case

```yaml
trigger: none
pr: none

resources:
  repositories:
    - repository: azure-devops-templates
      type: github
      endpoint: spydersoft-gh
      name: spydersoft-consulting/azure-devops-templates

variables:
  - group: atlassianVariables

extends:
  template: pipelines/extends/publish-md-to-confluence/v1.yml@azure-pipelines
  parameters:
    mdsToPublish:
      - path: docs/somedoc.md
        space: SK
        parent: "Test"
      - path: docs/subfolder/some-other-doc.md
        space: SK
        parent: "Test"
    confluenceOrg: $(orgName)
    confluenceUser: $(confluenceUser)
    confluenceApiKey: $(confluenceApiKey)
```

The `atlassianVariables` group contains the following variables:
| Name | Value |
| ---------------- | --------------------------------------------------- |
| orgName | The Atlassian cloud org name. |
| confluenceUser | The Atlassian username. Typcially an email address |
| confluenceApiKey | An API key tied to the Atlassian username |

## Parameters

| Name               | Type   | Description                                                                                                   | Default Value              |
| ------------------ | ------ | ------------------------------------------------------------------------------------------------------------- | -------------------------- |
| `confluenceOrg`    | string | The Atlassian cloud org name.                                                                                 |                            |
| `confluenceUser`   | string | he Atlassian username. Typcially an email address                                                             |                            |
| `confluenceApiKey` | string | An API key tied to the Atlassian username                                                                     |                            |
| `mdsToPublish`     | object | List Markdown files to publish Projects to Pack. See [Defining files to publish](#defining-files-to-publish). |                            |
| `repositoryName`   | string | Repository name, used as a label on the page                                                                  | `$(Build.Repository.Name)` |

### Defining files to publish

When passing files to publish, `mdsToPublish` expects an object with three properties.

| Name   | Value                                                                                |
| ------ | ------------------------------------------------------------------------------------ |
| path   | The relative path of the MD file to publish. DO NOT ADD `./` or `/` at the beginning |
| space  | The Space Key in Confluence to publish to                                            |
| parent | The name of the parent page to publish under                                         |

## Version History

### 1.0.0 \[v1.yml\]

- Initial Creation

[1]: https://github.com/mermaid-js/mermaid-cli "Mermaid CLI"
[2]: https://github.com/spydersoft-consulting/md_to_conf "md-to-conf"
