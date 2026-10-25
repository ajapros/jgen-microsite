---
title: "Configuration & versions"
eyebrow: "Getting started"
permalink: /configuration/
summary: "The generator config file, how ${…} placeholders resolve, and how sinceVersion gates a blueprint against the configured Chenile runtime."
---

## The config file

The default generator config is packaged at `jgen-base/src/main/resources/config.json`. It supplies the values
that most blueprints share, including:

- package segments: `com`, `company`, `org`
- the Chenile package (`chenilePackage`) and version (`chenileVersion`)
- a default service name, default version, and default destination folder

These config values are **merged into the runtime input map before template rendering**, so templates can
reference both blueprint inputs and global defaults. You can supply your own config with `-c`, and interactive
mode offers any config files it finds in a local `config/` folder.

## `${…}` placeholder substitution

A blueprint field's `defaultValue` can reference a config value with `${…}`:

```json
{
  "name": "destFolder",
  "defaultValue": "${defaultDestFolder}"
}
```

At runtime that placeholder is resolved from the selected config file before prompting or validation. The same
substitution works inside `super_pom` and dependency coordinate values; an unresolved Maven property reference
like `${project.version}` is left intact for Maven to resolve later.

## Runtime compatibility — `sinceVersion`

A blueprint can declare the minimum Chenile runtime its generated code needs with the optional `sinceVersion`
property. Use it when a template relies on an API, annotation, configuration option, or behaviour introduced in
a specific Chenile release.

```json
{
  "name": "chenile-headless-service",
  "description": "Generates a non-HTTP Chenile service",
  "sinceVersion": "2.1.31",
  "templateFolder": "headless-service-template"
}
```

The selected config supplies the installed-runtime target through `chenileVersion`:

```json
{ "chenilePackage": "org.chenile", "chenileVersion": "2.1.31" }
```

During generation JGen compares the two using **Maven version ordering**. Generation proceeds when the config
version is equal to or newer than `sinceVersion`. A blueprint with no `sinceVersion` is compatible with any
config.

| Blueprint `sinceVersion` | Config `chenileVersion` | Result |
| --- | --- | --- |
| omitted | `2.1.8` | generation proceeds |
| `2.1.31` | `2.1.8` | rejected |
| `2.1.31` | `2.1.31` | generation proceeds |
| `2.1.31` | `2.2.0` | generation proceeds |

When a check fails, the error names the blueprint, the required version, and the configured version:

```text
Blueprint 'chenile-headless-service' requires Chenile version 2.1.31 or later,
but the selected config declares 2.1.8.
```

### When the check runs

The check is an OWIZ processor named `version-validator`. It runs **after** JGen resolves the selected config
and **before** any blueprint hook, template copy, or generated-file change — so an incompatible generation
fails without producing partial output. The portal uses the same execution chain. Note that `jgen -g` only
emits a sample input file and does **not** run the generation chain; compatibility is enforced when that file is
later run with `jgen -f`.

### Choosing `sinceVersion` as an author

Set it to the first Chenile version that supplies every framework feature your template uses — not necessarily
the current JGen version. Leave it omitted when the template has no release-specific dependency. When a template
starts using a newer Chenile API, raise `sinceVersion` in the same change, and test generation against both the
minimum accepted config and the immediately older (rejected) config.
