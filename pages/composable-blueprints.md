---
title: "Composable blueprints"
eyebrow: "Authoring blueprints"
permalink: /composable-blueprints/
summary: "Every blueprint accepts an optional parent POM and extra dependencies, so generated projects slot under a shared Maven reactor without template edits."
---

Every built-in blueprint accepts `destFolder`, an optional `super_pom`, and an optional `dependencies` array.
These let a caller generate projects under a shared Maven parent and supply dependencies without changing
templates. Omitting them preserves the existing generated parent and dependencies.

## Supply a parent and dependencies

```json
{
  "blueprint": "chenile-service",
  "service": "orders",
  "serviceVersion": "1.0.0",
  "destFolder": "./output",
  "jpa": "n",
  "security": "n",
  "super_pom": {
    "groupId": "com.example",
    "artifactId": "application-parent",
    "version": "1.0.0",
    "relativePath": "../pom.xml"
  },
  "dependencies": [
    { "groupId": "com.example", "artifactId": "shared-contracts", "version": "1.0.0" }
  ]
}
```

The parent changes only in the **outermost** generated POM; child modules keep their generated project parent.
`relativePath` is relative to that outermost POM — not to the input file or `destFolder`.

The parent requires `groupId`, `artifactId` and `version`. If `relativePath` is omitted it defaults to
`../pom.xml`; an explicit empty string emits an empty `<relativePath>` so Maven resolves the parent from
repositories. An omitted, null, or empty-object `super_pom` keeps the default parent. A custom parent must
provide the dependency and plugin management the generated project expects — typically by inheriting
`org.chenile:chenile-parent` at the selected Chenile version.

Dependencies require `groupId` and `artifactId`; a `version` may be omitted when Maven dependency management
supplies it. Coordinate values support the usual `${…}` substitutions, and unresolved Maven property references
are retained.

## Declare the inputs in a blueprint

The type names are case-sensitive. Add these entries to `inputFields`:

```json
[
  { "name": "super_pom", "type": "SuperPOM", "description": "Optional parent POM", "required": false },
  { "name": "apiDependencies", "type": "POM_Dependencies", "description": "Extra deps for the API module", "required": false },
  { "name": "serviceDependencies", "type": "POM_Dependencies", "description": "Extra deps for the service module", "required": false }
]
```

`SuperPOM` and `POM_Dependencies` have built-in coordinate schemas — no `childFields` needed. Both are optional
by default; `required: true` demands a parent object, or at least one dependency, respectively. Each dependency
field is processed independently regardless of its name, and an omitted dependency field becomes an empty array.

**The blueprint owns placement.** Its POM template decides where the dependencies land:

```xml
<dependencies>
  {% raw %}{{#apiDependencies}}{% endraw %}
  <dependency>
    <groupId>{% raw %}{{groupId}}{% endraw %}</groupId>
    <artifactId>{% raw %}{{artifactId}}{% endraw %}</artifactId>
    {% raw %}{{#version}}<version>{{version}}</version>{{/version}}{% endraw %}
  </dependency>
  {% raw %}{{/apiDependencies}}{% endraw %}
</dependencies>
```

For `super_pom`, normalization also supplies `parentGroupId`, `parentArtifactId`, `parentVersion` and
`parentRelativePath`, which the built-in outer-POM templates use to select the supplied parent. New templates
can render the structured `super_pom` object directly, with an inverted section for the default parent. Do not
change generated child-module parent references.

## Built-in dependency destinations

Each built-in blueprint declares one `POM_Dependencies` field named `dependencies`, added alongside existing
dependencies at:

| Blueprint | Destination |
| --- | --- |
| `chenile-service` / `chenile-headless-service` / `wfservice` / `wfcustom` | `<service>/<service>-service/pom.xml` |
| `batch` | `<batch>/<batch>-service/pom.xml` |
| `chenile-process-workers` | `<workerModule>/pom.xml` |
| `chenile-process-management` | `<application>/<application>-server-service/pom.xml` |
| `minimonolith` | `<monolith>/<monolith>-package/pom.xml` |
| `mybatisQuery` | `<namespace>/pom.xml` |
| `chenile-interceptor` | `<interceptorName>/pom.xml` |
| `it` | `bdd-it-<app>/pom.xml` |
| `jgen-blueprint` | `bp-<blueprintName>/pom.xml` |
| `chenile-ecosystem` | Ecosystem root `pom.xml`; inherited by its generated children |

## CLI, portal, and programmatic callers

Input files use the JSON objects and arrays shown above. Interactive CLI mode asks whether to set a parent, then
captures its coordinates, and collects repeated dependency records. The portal exposes these types as JSON text
areas and publishes their coordinate fields in its blueprint schema. `jgen -g <blueprint> -o input.json` emits
`super_pom: null` and empty dependency arrays, which can be left unchanged to use the defaults. Programmatic
callers pass the same maps and lists through `BlueprintInputService.buildInputMap(...)` and
`BlueprintExecutor.execute(...)`; the execution boundary validates and normalizes these types before any hook or
file write. [Aggregated blueprints](/aggregated-blueprints/) use this same input contract when their callbacks
supply child inputs.

## Backward compatibility

The flat parent inputs remain supported: `parentGroupId`, `parentArtifactId`, `parentVersion`,
`parentRelativePath`. A structured `super_pom` takes precedence over these aliases. The legacy mini-monolith
dependency entry form (`dependencyGroup` / `dependencyName` / `dependencyVersion`) also still works; canonical
`groupId` / `artifactId` / `version` take precedence when both forms appear, and normalized entries keep both
key sets so existing template consumers keep working. The mini-monolith also accepts `deploymentUrl`,
`runtimeProperties` records, and optional `scanPackages` / `entityPackages` / `repositoryPackages` records —
persistence scans are emitted only in a JPA-enabled deployable host.
