# Publish MD to Confluence Template

## Purpose

This template publishes Markdown files from a repository into Confluence.

- **v1** — Publishes one file at a time. Uses the Mermaid CLI Docker image to pre-process diagrams before uploading.
- **v2** — Publishes multiple files per invocation with automatic cross-document link resolution. Mermaid rendering is handled natively by [md-to-conf][2] (no Docker required).

Both versions use [md-to-conf][2].

---

## v2 (Current)

### Features

- **Multi-file publishing** — files grouped by space/parent are published in a single `md-to-conf` call, so relative links between them (e.g. `[Guide](guide.md#section)`) are automatically rewritten to live Confluence URLs.
- **Glob patterns** — `paths` accepts shell glob patterns (`docs/*.md`, `docs/**/*.md`).
- **Exclusions** — each group supports an optional `exclude` list of glob patterns to skip specific files.
- **Mermaid rendering** — `--mermaid` flag is passed to `md-to-conf`, which renders diagrams using the local `mmdc` CLI if available, or falls back to the `mermaid.ink` public API. No Docker required.

### Usage

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
  template: pipelines/publish-md-to-confluence/v2.yml@azure-devops-templates
  parameters:
    confluenceOrg: $(orgName)
    confluenceUser: $(confluenceUser)
    confluenceApiKey: $(confluenceApiKey)
    publishGroups:
      # All files in a group are published together — links between them resolve
      - spaceKey: SK
        parent: "My Docs"
        paths:
          - docs/intro.md
          - docs/guide.md
          - docs/reference.md
      # A second group in a different space
      - spaceKey: OPS
        parent: "Runbooks"
        paths:
          - ops/runbook.md
```

#### With glob patterns and exclusions

```yaml
extends:
  template: pipelines/publish-md-to-confluence/v2.yml@azure-devops-templates
  parameters:
    confluenceOrg: $(orgName)
    confluenceUser: $(confluenceUser)
    confluenceApiKey: $(confluenceApiKey)
    publishGroups:
      - spaceKey: SK
        parent: "Documentation"
        paths:
          - docs/**/*.md
        exclude:
          - docs/drafts/**
          - docs/wip*.md
```

#### Disable Mermaid rendering

```yaml
extends:
  template: pipelines/publish-md-to-confluence/v2.yml@azure-devops-templates
  parameters:
    confluenceOrg: $(orgName)
    confluenceUser: $(confluenceUser)
    confluenceApiKey: $(confluenceApiKey)
    renderMermaid: false
    publishGroups:
      - spaceKey: SK
        parent: "My Docs"
        paths:
          - docs/intro.md
```

#### Use mermaid.ink fallback without installing mmdc

Useful when the agent has no npm, but does have outbound HTTPS:

```yaml
extends:
  template: pipelines/publish-md-to-confluence/v2.yml@azure-devops-templates
  parameters:
    confluenceOrg: $(orgName)
    confluenceUser: $(confluenceUser)
    confluenceApiKey: $(confluenceApiKey)
    installMermaidCli: false   # skip npm install; md-to-conf falls back to mermaid.ink
    publishGroups:
      - spaceKey: SK
        parent: "My Docs"
        paths:
          - docs/intro.md
```

### Parameters

| Name | Type | Description | Default |
| --- | --- | --- | --- |
| `confluenceOrg` | string | Atlassian cloud org name | |
| `confluenceUser` | string | Atlassian username (typically an email address) | |
| `confluenceApiKey` | string | API key tied to the Atlassian username | |
| `publishGroups` | object | List of publish groups. See [Defining publish groups](#defining-publish-groups). | `[]` |
| `repositoryName` | string | Repository name, used as a label on each published page | `$(Build.Repository.Name)` |
| `renderMermaid` | boolean | Render Mermaid code blocks as PNG images before uploading | `true` |
| `installMermaidCli` | boolean | Install `mmdc` (Mermaid CLI) via npm before publishing. Ignored when `renderMermaid` is `false`. Set to `false` if `mmdc` is already on the agent or you want to rely on the `mermaid.ink` API fallback. | `true` |

### Defining publish groups

Each entry in `publishGroups` maps to one `md-to-conf` invocation.

| Property | Required | Description |
| --- | --- | --- |
| `spaceKey` | ✅ | Confluence Space Key to publish to |
| `parent` | ✅ | Name of the parent page to publish under |
| `paths` | ✅ | List of file paths or glob patterns (relative to repo root) |
| `exclude` | | List of glob patterns to exclude from `paths` |

### Variable group

The `atlassianVariables` variable group should contain:

| Name | Value |
| --- | --- |
| `orgName` | The Atlassian cloud org name |
| `confluenceUser` | The Atlassian username (typically an email address) |
| `confluenceApiKey` | An API key tied to the Atlassian username |

### Migrating from v1

**Before (v1)** — each file is a separate entry in a flat `mdsToPublish` list:

```yaml
extends:
  template: pipelines/publish-md-to-confluence/v1.yml@azure-devops-templates
  parameters:
    confluenceOrg: $(orgName)
    confluenceUser: $(confluenceUser)
    confluenceApiKey: $(confluenceApiKey)
    mdsToPublish:
      - path: docs/intro.md
        space: SK
        parent: "My Docs"
      - path: docs/guide.md
        space: SK
        parent: "My Docs"
```

**After (v2)** — files sharing the same space and parent are grouped:

```yaml
extends:
  template: pipelines/publish-md-to-confluence/v2.yml@azure-devops-templates
  parameters:
    confluenceOrg: $(orgName)
    confluenceUser: $(confluenceUser)
    confluenceApiKey: $(confluenceApiKey)
    publishGroups:
      - spaceKey: SK
        parent: "My Docs"
        paths:
          - docs/intro.md
          - docs/guide.md
```

Key differences:
- `mdsToPublish` → `publishGroups`
- `space` → `spaceKey`
- `path` (single string) → `paths` (list, supports globs)
- Docker is no longer needed — remove any Docker socket mounts or Docker-in-Docker setup
- The `$(Build.ArtifactStagingDirectory)` staging step is no longer needed

---

## v1 (Legacy)

### Purpose

Publishes individual Markdown files to Confluence, one file per `md-to-conf` call. Mermaid diagrams are pre-processed via the [Mermaid CLI][1] Docker image before upload.

### Usage

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
  template: pipelines/publish-md-to-confluence/v1.yml@azure-devops-templates
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

### Parameters

| Name | Type | Description | Default |
| --- | --- | --- | --- |
| `confluenceOrg` | string | Atlassian cloud org name | |
| `confluenceUser` | string | Atlassian username (typically an email address) | |
| `confluenceApiKey` | string | API key tied to the Atlassian username | |
| `mdsToPublish` | object | List of Markdown files to publish. See [Defining files to publish](#defining-files-to-publish-v1). | |
| `repositoryName` | string | Repository name, used as a label on each published page | `$(Build.Repository.Name)` |

### Defining files to publish (v1)

| Property | Description |
| --- | --- |
| `path` | Relative path of the Markdown file from repo root. Do not prefix with `./` or `/` |
| `space` | Confluence Space Key to publish to |
| `parent` | Name of the parent page to publish under |

---

## Version History

### 2.0.0 \[v2.yml\]

- Multi-file publishing with automatic cross-document link resolution
- Glob pattern support for `paths`
- Exclusion support via `exclude` glob patterns per group
- Mermaid rendering handled natively by `md-to-conf` (`--mermaid`); `mmdc` installed via npm when `renderMermaid: true` (opt-out with `installMermaidCli: false` to use the `mermaid.ink` API fallback instead). Docker no longer required.
- Parameters restructured: `mdsToPublish` replaced by `publishGroups`

### 1.0.0 \[v1.yml\]

- Initial creation

[1]: https://github.com/mermaid-js/mermaid-cli "Mermaid CLI"
[2]: https://github.com/spydersoft-consulting/md_to_conf "md-to-conf"
