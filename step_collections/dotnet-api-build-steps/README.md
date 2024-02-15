# .Net API Build Steps

## Purpose

This template contains common steps for building, testing, and analyzing .NET API projects.

## Usage

### Use Case

This is just a step template, so it needs to be included in steps as part of your stage and job. Below is a simple example

```yaml

```

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
