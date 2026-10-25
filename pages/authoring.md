---
title: "Write your own blueprint"
eyebrow: "Authoring blueprints"
permalink: /authoring/
summary: "Scaffold a new bp-* module with jgen-blueprint, add its template tree, declare its inputs, write its init hook, and wire it into the build."
---

A blueprint is a small Maven module with three parts: a `blueprint.json` descriptor (its public contract), a
template tree (what it generates), and an optional init hook (derived values). The `jgen-blueprint` blueprint
scaffolds the shell; you finish the template and hook by hand.

## Step 1 — Scaffold the module

Build the repo, emit a sample input, edit it, and run:

```bash
make all
jgen -g jgen-blueprint -o new-blueprint.json
```

```json
{
  "blueprint": "jgen-blueprint",
  "blueprintName": "invoice",
  "destFolder": "./output",
  "security": "n",
  "jpa": "y",
  "cloudSwitchEnabled": "n"
}
```

```bash
jgen -f new-blueprint.json
```

That produces `./output/bp-invoice` with the usual layout:

```text
bp-invoice/
  pom.xml
  src/main/java/org/chenile/jgen/blueprint/invoice/package-info.java
  src/main/java/org/chenile/jgen/blueprint/invoice/InitInvoiceBlueprint.java
  src/main/resources/META-INF/blueprint.json
```

This is only a starting point — a module, a descriptor, an init stub, and a package. It does **not** include the
template tree that defines what your blueprint generates.

## Step 2 — Add the template tree

Look at the `templateFolder` declared in the generated `blueprint.json` (for example `invoice-template`), then
create that folder under `src/main/resources/` and put the file/directory structure your blueprint should
generate inside it:

```text
src/main/resources/invoice-template/
  __invoice__/
    __invoice__-service/...
```

Use the [template conventions](/template-conventions/) — `.mustache` files, `__name__` path placeholders,
`%%key=value%%` conditional folders, `.filelist` multi-file outputs. The existing `bp-service` and
`bp-wfservice` modules are good references.

## Step 3 — Declare the inputs

The generated `blueprint.json` is the CLI contract: name, description, template folder, and the fields the user
supplies. Edit the `inputFields` to match your domain. A field declares a `name`, a `type` (`STRING`,
`BOOLEAN`, `FILE`, `DROPDOWN` + `validValues`, `RECORD_ARRAY` + `childFields`, `SuperPOM`, `POM_Dependencies`),
an optional `defaultValue`, `required`, a [`validation`](/input-validation/) rule, and `visibleWhen`:

```json
{
  "name": "invoice",
  "type": "STRING",
  "validation": "JAVA_IDENTIFIER",
  "description": "Java identifier for the invoice service",
  "required": true
}
```

If your blueprint reads an external definition file, add a `FILE` field. If its generated code relies on a
Chenile feature introduced after the oldest supported release, add a top-level `sinceVersion` — JGen will then
reject generation (before the hook or any copy) for a config whose `chenileVersion` is too old.

## Step 4 — Write the init hook

The generated init class is a stub. Its job is to populate derived values in the template map **before**
generation — capitalized names, derived package fragments, parsed XML/JSON, or post-generation logic. Register a
`postInputCaptureHook` in `init`:

```java
public void init(BlueprintConfig blueprintConfig) {
    blueprintConfig.postInputCaptureHook = (Map<String,Object> map) -> {
        String invoice = (String) map.get("invoice");
        map.put("Invoice", CapUtils.capitalizeFirst(invoice));
    };
}
```

> The scaffold's init stub does not always line up with the generated `blueprint.json`. Review it right after
> generation and make sure it reads the correct field names from the map.

For heavier work, study the built-ins:

- `bp-service` — simple field derivation (`service` → `Service`).
- `bp-wfcustom` — parses a workflow XML file and enriches the map with workflow/testcase data.
- `bp-batch` — file-driven preprocessing plus post-generation copy behaviour.

A blueprint that composes other blueprints also registers `childBlueprintInputs` here — see
[Aggregated blueprints](/aggregated-blueprints/). A blueprint that emits companion sample files registers
`sampleInputFiles` here too — see [the CLI page](/install/#3-emit-a-sample-input-and-its-companion-files).

## Step 5 — Use the template conventions

Your templates get the same conventions as the built-ins because they are implemented by the shared processor
chain: `.mustache` contents, `__name__` paths, `%%key=value%%` folders, and `.filelist` outputs. Nothing extra
to enable.

## Step 6 — Wire the module into the build

Discovery is classpath-based, so the CLI sees your blueprint only if its module is on the CLI classpath. Update
the root `pom.xml` (add the new module under `<modules>`, and under `<dependencyManagement>` if you want it
managed like the others), and add the module as a dependency in `jgen-cli/pom.xml`.

## Step 7 — Test and iterate

```bash
make all                          # rebuild with the new module on the classpath
jgen -g invoice -o invoice.json   # emit a sample for your blueprint
jgen -f invoice.json              # generate
```

Then inspect the output tree and iterate on the template folder and hook. The fastest way to debug a blueprint:

1. confirm the input map is what you expect;
2. confirm the template resource tree matches the intended output shape;
3. check conditional folders and placeholder names;
4. inspect the generated output tree after one run.

## Where the real logic lives

`jgen-base` defines the framework, `jgen-cli` handles interaction and input files, and the blueprint modules
define the actual output. In practice most user-visible changes are **template edits**, not Java changes. If you
want to change *global* generation behaviour, start in `jgen-base`; if a generated project is wrong for just one
blueprint, inspect that blueprint's templates first.
