---
title: "Install & the CLI"
eyebrow: "Getting started"
permalink: /install/
summary: "Build JGen, put it on your PATH, and learn the CLI flags and the interactive, file-based, and sample-emitting flows."
---

## Build and set up

```bash
git clone https://github.com/rajakolluru/chenile-gen.git && cd chenile-gen
make clean all && source setpath.sh          # builds the modules, puts jgen and stm-cli on PATH
```

`make all` builds the full multi-module Maven project and prepares the CLI wrapper script. The non-PATH
equivalent of the `jgen` command is `chenile-gen/jgen/jgen-cli/bin/jgen.sh`, which finds the latest built
`jgen-cli-*.jar` under `target/` and runs it with `java -jar`.

Generated projects build with `mvn install` (or `make build`); generated mini-monoliths run with `make run`
(port 8080 by default).

## CLI options

| Flag | Meaning |
| --- | --- |
| `-c, --config` | Load a config JSON instead of the bundled default config. |
| `-e, --emit-config` | Copy global config into `config/`; with `-g`, also emit companion sample input files. |
| `-f, --input-file` | Read blueprint inputs from a JSON file (non-interactive). |
| `-g, --generate-blueprint-file` | Emit a sample input JSON for a named blueprint. |
| `-o, --output-file` | Where to write the sample input JSON. |

There are three ways to drive JGen.

## 1. Non-interactive — generate from a file

The safest and most repeatable flow. First emit a sample input contract for a blueprint, then edit it, then run
it:

```bash
jgen -g chenile-service -o orders.json      # 1. emit a sample
# 2. edit orders.json
jgen -f orders.json                         # 3. generate
```

`-g` is the safest way to learn a blueprint, because the emitted JSON shows exactly which fields that blueprint
expects. A typical input file for `chenile-service`:

```json
{
  "blueprint": "chenile-service",
  "service": "orders",
  "serviceVersion": "0.0.1-SNAPSHOT",
  "destFolder": "./output",
  "security": "n",
  "jpa": "y",
  "cloudSwitchEnabled": "n"
}
```

The `blueprint` field selects the blueprint; JGen fills missing values from defaults, validates the result, and
generates. Booleans are supplied in the CLI's own `y` / `n` form; an enabled boolean is normalized to the string
`"true"` in the template map.

## 2. Interactive

Running `jgen` with no `-f` starts interactive mode:

1. If a local `config/` folder exists, the CLI offers those config files as choices.
2. It loads all registered blueprints from the classpath and shows them as a menu.
3. It prompts for each declared input field.
4. It runs the selected blueprint through the shared pipeline.

Pressing enter on a prompt accepts the default when the field has one; a field with no default is required
before generation can continue. Boolean fields accept `y` or `n`, and `FILE` fields must point to an existing
file. Defaults such as `${defaultVersion}` are resolved against the chosen config before prompting.

## 3. Emit a sample input (and its companion files)

`-g` alone writes a placeholder sample JSON. Adding `-e <folder>` additionally emits any **blueprint-owned
companion files** — the external spec files that a `FILE` input points at:

```bash
jgen -g chenile-process-management -e process-specs
jgen -g wfcustom -e workflow-specs
```

The first command creates `process-specs/input-file.json` plus `process-specs/input-file-processSpec.json` and
`process-specs/input-file-cronSpec.json`; the second creates `workflow-specs/input-file.json` and
`workflow-specs/input-file-xmlFile.xml`. The main input JSON already references the companions by path — fill in
the project/service names, customize the samples, then run `jgen -f process-specs/input-file.json`.

Emission never runs project-generation hooks and refuses to overwrite an existing main or companion file.
Standalone `jgen -e <config-file>` (no `-g`) still emits global configuration.

> Blueprint authors register these emitters in `BlueprintConfig.sampleInputFiles`, keyed by the FILE input name.
> The emitter owns the content; the framework assigns the filename `<stem>-<field><ext>`, invokes the emitter and
> fills the input reference. Packaged resources use `SampleInputFile.resource(...)`; custom emitters can write
> their own sample data without any CLI change.

## A worked example

```bash
make all
jgen -g chenile-service -o orders.json
# edit orders.json → service: "orders", jpa: "y"
jgen -f orders.json
```

What happens on that last command:

1. the service blueprint is selected from the registry;
2. its init hook derives `Service=Orders`;
3. the `service-template` tree is copied into `./output`;
4. `.mustache` files are rendered from the input/config map;
5. path placeholders such as `__service__` and `__Service__` are renamed;
6. conditional folders such as `%%jpa=true%%` are applied;
7. the result is a ready-to-build `orders` service project.

See [the generation pipeline](/pipeline/) for the full ordered list of steps.
