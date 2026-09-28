# .NET Multi-Publish Build Steps

## Purpose

Common steps for building a solution containing multiple publishable .NET projects
(e.g. an API and a background processor sharing one repo), running its tests, running
Sonar analysis, and publishing each project as its own pipeline artifact. Used by
[build-multi-container](/pipelines/build-multi-container/v1.yml), which adds the
docker/helm/nuget stages around this step collection.

## Usage

This is a step template, so it needs to be included in steps as part of your stage and
job.

```yaml
steps:
  - template: /step_collections/dotnet-multi-publish-build-steps/v1.yml
    parameters:
      buildProject: src/MySolution.slnx
      publishProjects:
        - name: api
          project: src/MyApi/MyApi.csproj
          artifactName: api
      executeTests: true
      testProjects: "src/**/*.UnitTests/*.csproj"
```

### Parameters

| Name                      | Type   | Description                                                                                                                                               | Default Value                            |
| ------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `buildConfiguration`      | string | Build configuration.                                                                                                                                      | `Release`                                |
| `netCoreVersion`          | string | .NET SDK version. See [UseDotNet@2][1] documentation for syntax.                                                                                          | `8.0.x`                                  |
| `buildProject`            | string | The solution (or csproj glob) to build. Building the solution is what makes the SonarQube/SonarCloud scanner see all C# projects as one Sonar project.    | `**/*.slnx`                              |
| `publishProjects`         | object | List of projects to publish, each becomes its own pipeline artifact. See [Publish Projects](#publish-projects).                                           | `[]`                                     |
| `externalFeedCredentials` | string | NuGet service connection name used for `dotnet restore`.                                                                                                  | `SpydersoftGithub`                       |
| `nugetConfigPath`         | string | Path to `nuget.config`. Override for repos that keep it somewhere other than the checkout root (e.g. a monorepo with it under a shared `src/` directory). | `$(Build.SourcesDirectory)/nuget.config` |

### Test Parameters

| Name           | Type     | Description                                                                                                                                                                                              | Default Value        |
| -------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| `executeTests` | boolean  | If `true`, tests execute after the build.                                                                                                                                                                | `false`              |
| `testProjects` | string   | Projects to test. See [file matching patterns reference][3]. Only used by `testRunner: vstest` — `mtp` discovers test projects itself from `buildProject` (see [Test Runner Modes](#test-runner-modes)). | `**/*tests/*.csproj` |
| `testRunner`   | string   | `vstest` or `mtp`. See [Test Runner Modes](#test-runner-modes).                                                                                                                                          | `vstest`             |
| `preTestSteps` | stepList | Steps executed immediately before the test step.                                                                                                                                                         | `[]`                 |

### Sonar Parameters

A single Sonar scope wraps the whole solution build, so all C# code reports under one
Sonar project.

| Name                     | Type    | Description                                                                                                                                      | Default Value |
| ------------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------- |
| `executeSonar`           | boolean | If `true`, runs Sonar prepare/analyze/publish around the build.                                                                                  | `false`       |
| `sonarEndpointName`      | string  | Azure DevOps service connection name for SonarQube/SonarCloud. Required when `executeSonar=true`.                                                | `""`          |
| `sonarProjectKey`        | string  | SonarQube/SonarCloud project key. Required when `executeSonar=true`.                                                                             | `""`          |
| `sonarProjectName`       | string  | SonarQube/SonarCloud project name. Required when `executeSonar=true`.                                                                            | `""`          |
| `useSonarCloud`          | boolean | `true` for SonarCloud, `false` for SonarQube (on-prem).                                                                                          | `false`       |
| `sonarCloudOrganization` | string  | SonarCloud organization name. Required when `useSonarCloud=true`.                                                                                | `""`          |
| `sonarExtraProperties`   | string  | Extra properties appended to the Sonar `extraProperties` block (both coverage reportsPaths properties below are always included ahead of these). | `""`          |

### Hooks

| Name            | Type     | Description                             | Default Value |
| --------------- | -------- | --------------------------------------- | ------------- |
| `prebuildSteps` | stepList | Steps executed before `dotnet restore`. | `[]`          |

## Publish Projects

Each entry in `publishProjects` is published to its own staging subdirectory and
uploaded as its own pipeline artifact (consumed by a downstream docker/helm stage).

| Field          | Description                                                                  |
| -------------- | ---------------------------------------------------------------------------- |
| `name`         | Short identifier — used for the staging subdirectory and step display names. |
| `project`      | Path/glob to the csproj to publish.                                          |
| `artifactName` | Pipeline artifact name.                                                      |

## Test Runner Modes

| Mode               | Test host                                                                                                                      | Coverage                                                                                                 | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `vstest` (default) | Classic VSTest, via `DotNetCoreCLI@2`'s `command: test`.                                                                       | `coverlet.collector` (`coverage.opencover.xml` + `coverage.cobertura.xml`).                              | Works for any test project (MSTest/NUnit/xUnit) that hasn't opted into Microsoft.Testing.Platform (MTP).                                                                                                                                                                                                                                                                                                                                           |
| `mtp`              | [Microsoft.Testing.Platform][5]-hosted test projects, via a one-line `pwsh` step that calls `dotnet test --solution` directly. | [`Microsoft.Testing.Extensions.CodeCoverage`][6] (`coverage.cobertura.xml` only — no OpenCover support). | Required once a repo's test projects enable their framework's MTP runner (e.g. NUnit's `EnableNUnitRunner`) and pull in `Microsoft.Testing.Platform.MSBuild` 2.x: VSTest-mode `dotnet test` becomes a hard build error for those projects on .NET 10 SDK. Each test project must also reference `Microsoft.Testing.Extensions.TrxReport` and `Microsoft.Testing.Extensions.CodeCoverage` — `coverlet.collector` doesn't hook into MTP-hosted runs. |

`vstest` mode uses `DotNetCoreCLI@2`'s `command: test` as normal. `mtp` mode can't: that
input unconditionally shells out to `MSBuild --target:VSTest` regardless of `arguments`
(confirmed via a real CI failure — `MSB1001: Unknown switch` on `--report-trx`), and a
repo's `global.json` `test.runner` opt-in only takes effect through the real `dotnet test`
CLI. So `mtp` mode calls `dotnet test --solution ${{ parameters.buildProject }}` directly
instead — the CLI does its own test-project discovery (via `IsTestProject`, same as the
SDK itself), so no project/module enumeration is needed at all; `testProjects` is unused
in this mode. `--max-parallel-test-modules 1` is the one non-obvious flag: `--coverage-output`
is a fixed filename, and MTP's own results-subfolder naming for that isn't reliably
collision-free when modules run in parallel sharing one `--results-directory` (a coverage
report was silently dropped without it during validation) — serializing avoids the
collision at some cost to wall-clock time.

Both modes share the same "Copy Test Files" / `PublishTestResults@2` / `PublishCodeCoverageResults@2`
steps: trx and coverage files are searched for recursively under `$(Agent.TempDirectory)`
(covering both VSTest's layout and MTP's per-run, machine+timestamp-named results
subfolder) and deduped into `ResultFiles/<guid>/` before publishing. The Sonar
`extraProperties` block always includes both `sonar.cs.opencover.reportsPaths` and
`sonar.cs.cobertura.reportsPaths` — whichever glob matches real files is the one that
contributes; the other is a harmless no-op.

## Implementation Notes

- **Every `--no-build` step** (test, and each `publishProjects` publish) reuses the
  single versioned solution build done earlier in this template. Rebuilding at any of
  those points would happen without the `InformationalVersion`/`AssemblyVersion` MSBuild
  properties that build passes, regenerating `AssemblyInfo` at the default `1.0.0.0` and
  clobbering shared transitive dependencies in `bin/` — a later `--no-build` step would
  then ship those at the wrong version next to a correctly-versioned entry assembly and
  fail to load them at runtime (`FileNotFoundException`). Publishing this way also
  ensures every published artifact reflects the exact bits Sonar scanned.
- `publishWebProjects: false` on each publish step: it defaults to `true`, which makes
  the task ignore its own `projects` input and auto-glob for web-SDK csprojs instead —
  disabled so the explicit per-artifact `project` always wins.

## Version History

### 1.0.0 \[v1.yml\]

- Initial Creation

### 1.1.0 \[v1.yml\]

- Added `testRunner` parameter (`vstest` / `mtp`) for repos whose test projects have
  migrated to Microsoft.Testing.Platform.

### 1.1.1 \[v1.yml\]

- `mtp` mode simplified to a single `dotnet test --solution` call.

[1]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/use-dotnet-v2?view=azure-pipelines "UseDotNet@2 Documentation"
[3]: https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/file-matching-patterns?view=azure-devops "File matching patterns reference"
[5]: https://learn.microsoft.com/en-us/dotnet/core/testing/microsoft-testing-platform-intro "Microsoft.Testing.Platform overview"
[6]: https://github.com/microsoft/codecoverage "Microsoft.Testing.Extensions.CodeCoverage"
