---
title: "Generation pipeline"
eyebrow: "Authoring blueprints"
permalink: /pipeline/
summary: "The ordered processor chain in jgen-base that turns a selected blueprint and an input map into a generated project tree."
---

Generation is a fixed, data-driven chain of processors defined in
`jgen-base/src/main/resources/org/chenile/jgen/template/processors.xml`. The CLI does selection and validation;
this chain does all of the file work. Understanding the order is the key to understanding why a template behaves
the way it does — each step sees the tree as the previous step left it.

## The execution path in code

1. `GenMain` prepares the `Map<String,Object>` input map (merging inputs and config).
2. `BlueprintExecutor` wraps that map in a `TemplateContext`.
3. The orchestration engine runs the processors listed in `processors.xml`.
4. Each processor mutates the output tree or the context.

The portal uses this same chain, so CLI and portal generation behave identically.

## The processor chain, in order

The `processors` flow runs these thirteen processors, in this sequence:

| # | Processor | What it does |
| --- | --- | --- |
| 1 | `config-defaulter` | Enhances the input map with values from the selected config file. |
| 2 | `version-validator` | Rejects a blueprint whose `sinceVersion` is newer than the config's `chenileVersion` (see [Configuration](/configuration/)). |
| 3 | `generic-input-validator` | Validates common inputs — the [declarative rules](/input-validation/) — before any hook or file write. |
| 4 | `post-input-executor` | Runs the blueprint's **post-input (init) hook**, letting the author manipulate the input map (derive names, parse files). |
| 5 | `folder-copier` | Copies the blueprint's template folder from its JAR into the destination. |
| 6 | `lambda-loader` | Loads helper lambda functions that templates can call. |
| 7 | `file-template-executor` | Renders every `.mustache` file to its destination file. |
| 8 | `file-splitter` | Breaks every `.filelist` file into multiple files. |
| 9 | `file-name-replacer` | Replaces `__name__` placeholders in file and folder names from the map. |
| 10 | `conditional-processor` | Processes `%%key=value%%` folders: promote on match, discard otherwise. |
| 11 | `blueprint-expander` | Expands surviving child markers (`%{{…}}`) through their own pipelines — see [Aggregated blueprints](/aggregated-blueprints/). |
| 12 | `post-process-executor` | Runs the blueprint's **post-process hook** for custom destination-folder logic, after all children. |

> The engine resolves config, then validates version, then validates inputs — all **before** the init hook runs
> or a single file is copied. An incompatible or invalid generation fails without leaving partial output. Child
> blueprints are expanded late (step 11), after the parent's own conditional folders have been selected, so a
> child inside an excluded `%%…%%` folder is never resolved or invoked.

## What this means for authors

- Your **init hook** (step 4) runs after validation but before copying — so it can rely on validated inputs and
  safely add derived values the templates need.
- Placeholder renaming (step 9) happens **after** content rendering (step 7); a Mustache expression inside a
  file and a `__placeholder__` in its path are resolved at different stages, from the same map.
- Conditional selection (step 10) happens **before** child expansion (step 11); wrap an aggregation marker in a
  `%%flag=true%%` folder to include a child conditionally.
- Your **post-process hook** (step 12) runs last, once every child has been generated.

See [Write your own blueprint](/authoring/) for how to register the init and post-process hooks.
