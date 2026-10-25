---
title: "Overview & architecture"
eyebrow: "Getting started"
permalink: /overview/
summary: "What JGen is, the three parts it is made of, and how a blueprint turns a JSON input into a generated project."
---

JGen is a Java-based code generator for the Chenile ecosystem. It packages a shared generation **engine**, a
**CLI**, and a set of **blueprint** modules that each generate a different kind of Chenile project. It generates
application code; it is not the runtime. (The older `app-gen` / `gen.sh` generator is deprecated — always use
JGen.)

At a high level, every generation runs the same four beats:

1. The CLI loads blueprint plugins from the classpath.
2. You select a blueprint and provide inputs (interactively or from a JSON file).
3. The engine copies that blueprint's template tree into an output folder.
4. The engine renders the templates, renames placeholder paths, applies conditional folders, expands any
   included child blueprints, and runs the blueprint's hooks.

## The three parts

| Module | Responsibility |
| --- | --- |
| `jgen-base` | The core generation engine: blueprint model, config loading, the processor pipeline, and all the file-processing steps. Change this to change *global* generation behaviour. |
| `jgen-cli` | The command-line entry point. It loads config, discovers blueprints, collects and validates input, and hands the selected blueprint and input map to the engine. It does very little generation work itself. |
| `bp-*` | The blueprint plugins. Each one contributes a `META-INF/blueprint.json` descriptor, an optional `Init…Blueprint` hook, and a template folder. This is where the actual generated output is defined. |

The design is intentionally **data-driven**: the CLI does selection and validation, the processor chain does the
file work, and most of a blueprint's behaviour is encoded in its template tree rather than in Java. If a
generated file looks wrong for one blueprint, the first place to look is that blueprint's template folder.

## Anatomy of a blueprint

A blueprint module follows one shape:

```text
bp-something/
  src/main/java/.../InitSomethingBlueprint.java      # optional, imperative: derived values
  src/main/resources/META-INF/blueprint.json         # declarative: the public contract
  src/main/resources/something-template/...           # the template tree that gets generated
```

The JSON descriptor is the blueprint's public contract. It declares:

- `name` — the value used with `-g` and in the `blueprint` field of an input file
- `description`
- optional `sinceVersion` — the minimum Chenile runtime the generated code needs (see [Configuration](/configuration/))
- `templateFolder` — the resource folder copied and rendered
- `inputFields` — the fields the user must supply, with types, defaults, and validation rules
- an optional `initHook`

The Java init hook is imperative. Its job is to add computed values to the template map before generation —
capitalized names, derived packages, or structures parsed from an input file. Some blueprints do only trivial
derivation (`service` → `Service`); others parse a workflow XML or a batch JSON and inject a whole structure.

## How blueprints plug in

Blueprint discovery is **classpath-based**. The registry in `jgen-base` scans the classpath for every
`META-INF/blueprint.json`, deserializes each into a blueprint definition, and invokes its optional init hook. A
blueprint is therefore available to the CLI only if its module is on the CLI classpath — which is why adding a
new blueprint means wiring its module into the build (covered in [Write your own blueprint](/authoring/)).

## Where to go next

- [Install &amp; the CLI](/install/) — build it, put `jgen` on your PATH, and learn the flags and flows.
- [Configuration &amp; versions](/configuration/) — the config file, `${…}` substitution, and `sinceVersion` compatibility.
- [Blueprint catalogue](/blueprints/) — all fourteen built-in blueprints and their inputs.
- [Write your own blueprint](/authoring/) — scaffold, template, hook, and wire in a new `bp-*` module.
