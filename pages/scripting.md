---
title: "Scripting hooks (JavaScript)"
eyebrow: "Authoring blueprints"
permalink: /scripting/
summary: "Write a blueprint's hooks in JavaScript instead of Java. Scripts run in a sandbox, need no build, and work in template-only blueprints."
---

<span class="pill">2.5.1</span>

A blueprint's hooks compute derived values, shape the inputs of child blueprints and tidy the generated files.
Until now they were Java, so a blueprint with any logic needed a build and a jar. Hooks can now also be
**JavaScript files** shipped next to the template. JGen runs them with GraalJS, which is built into JGen, so there
is nothing extra to install.

Because scripts need no compiling, a blueprint with scripts can still be a
[template-only blueprint](/remote-blueprints/): a Git repository with a tag, used straight from GitHub or from
`dir:` while you write it.

## Declaring scripts

List them in `META-INF/blueprint.json` under `scripts`, by hook name. Paths are resources of the blueprint (inside
its jar or folder):

```json
{
  "name": "terraform",
  "templateFolder": "terraform-template",
  "scripts": {
    "postInputCapture": "scripts/hooks.js",
    "postProcess": "scripts/hooks.js"
  },
  "inputFields": [ ... ]
}
```

One file can hold several hooks. Each hook needs a function of its name in the file it points to:

| Hook | Function | Runs | Returns |
|---|---|---|---|
| `postInputCapture` | `postInputCapture(inputs, jgen)` | after inputs are validated, before any copy | an object whose entries are added to the inputs |
| `childBlueprintInputs` | `childBlueprintInputs(inputs, child, attributes, jgen)` | for each {% raw %}`%{{child}}`{% endraw %} marker of an aggregated blueprint | the child's inputs |
| `postProcess` | `postProcess(inputs, project, jgen)` | after the files are generated | nothing |
| `final` | `final(inputs, project, jgen)` | last, after the generation record and Git | nothing |

`inputs` and the returned objects are plain JSON values. Java hooks and scripts can be mixed: a script runs after
the Java hook of the same kind, so it sees what the Java hook added. Only `childBlueprintInputs` must come from one
place.

## An example

```javascript
// scripts/hooks.js
function postInputCapture(inputs, jgen) {
  return {
    Module: jgen.pascalCase(inputs.module),           // "network edge" -> "NetworkEdge"
    moduleVar: jgen.camelCase(inputs.module),         // "network edge" -> "networkEdge"
    regions: inputs.multiRegion ? ["us-east-1", "eu-west-1"] : ["us-east-1"]
  };
}

function postProcess(inputs, project, jgen) {
  if (!inputs.withExamples) project.delete("examples");
  project.write("VERSION", inputs.version + "\n");
  jgen.log("generated " + project.name());
}
```

The templates can then use {% raw %}`{{Module}}`{% endraw %}, `__moduleVar__` and {% raw %}`{{#regions}}…{{/regions}}`{% endraw %} as with any other input.

## What scripts can use

**`jgen`** — helpers:

- `capitalize(text)`, `uncapitalize(text)`, `camelCase(text)`, `pascalCase(text)`, `snakeCase(text)`,
  `upperSnakeCase(text)`, `kebabCase(text)`, and `helper(name, text)` for any
  [template helper](/extensions/#template-helpers), including language packs' helpers
- `log(message)` prints a line, prefixed with the blueprint's name
- `readInput(field)` returns the text of a `FILE` input the blueprint declares

**`project`** — the generated project's folder (`postProcess` and `final` only). Every path is relative to it:

- `name()`, `exists(path)`, `read(path)`, `write(path, text)` (creates folders), `move(from, to)`,
  `delete(path)` (files or whole folders, never the project itself), `list(path)`
- `copyInput(field, path)` copies a declared `FILE` input into the project

Standard JavaScript (ES2023: `JSON`, `Math`, regular expressions, string and array methods) is all there.

## The sandbox

A blueprint from a catalog or from GitHub is someone else's code, so scripts are confined:

- **No Java.** `Java.type` and every Java object except the methods above are refused.
- **No files outside the project.** `project` rejects any path that leaves the generated folder (`../`, absolute
  paths). There is no other file access.
- **No network, processes, threads, native code or environment variables.**
- **A statement limit** stops runaway loops with an error instead of a hung `jgen`.
- **Each call starts fresh.** Nothing is shared between hooks or between runs; pass values along in the inputs.

Scripts are compiled when the blueprint loads. A syntax error, a missing function or an unknown hook name is a
blueprint error reported before any prompt. A failure while running names the script, the hook and the line.

Trust still applies: a remote blueprint with scripts needs the same trust as any other, and its hash covers the
scripts.

## When to use Java instead

Scripts cover derived names, conditional values, child inputs and file tidying, which is what most hooks do. Use a
Java `initHook` (and publish a jar) when the hook needs a Java library, reads files outside the project, or talks
to a service.
