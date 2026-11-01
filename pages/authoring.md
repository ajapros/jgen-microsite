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

Emit a sample input, edit it, and run it:

```bash
jgen -g jgen-blueprint -o new-blueprint.json
```

```json
{
  "blueprint": "jgen-blueprint",
  "blueprintName": "invoice",
  "destFolder": "./output",
  "standaloneRepo": "y",
  "groupId": "com.acme",
  "blueprintVersion": "0.1.0",
  "githubRepository": "acme/jgen-invoice",
  "security": "n",
  "jpa": "y",
  "cloudSwitchEnabled": "n"
}
```

```bash
jgen -f new-blueprint.json
```

`standaloneRepo` (2.5.1) decides where the blueprint will live:

- **`y`: its own repository** (recommended for anything you will share). `./output/bp-invoice` is a
  self-contained Maven project:
  - its own `pom.xml`, with `org.chenile.jgen:jgen-base` (the generating JGen's version) as a provided dependency;
  - a `README.md`;
  - a `.gitignore`;
  - a `.github/workflows/release.yml` that publishes the jar when you push a version tag.

  It is already a **local Git repository** on branch `main` with one commit: `jgen-blueprint` sets `initGit` for
  standalone projects and names `githubRepository` as its `gitRepositoryField` (see
  [Generating a Git repository](#generating-a-git-repository-initgit)). JGen prints the remaining steps:

  ```text
  Git repository ready: /…/output/bp-invoice (branch main, 1 commit)
  Remote origin: git@github.com:acme/jgen-invoice.git
  Next:
    1. Create an empty repository on GitHub: acme/jgen-invoice (no README, licence or .gitignore)
    2. git -C /…/output/bp-invoice push -u origin main
  Release it (after pushing): git -C /…/output/bp-invoice tag 0.1.0 && git -C /…/output/bp-invoice push origin 0.1.0
  Then anyone can use it with: jgen -g github:acme/jgen-invoice
  ```

  `groupId`, `blueprintVersion` and `githubRepository` are asked only when `standaloneRepo` is `y`.
  `githubRepository` is optional. When given, it fills in the README and becomes the `origin` remote; nothing is
  pushed for you. If Git is not installed, or has no `user.name` and `user.email`, JGen still generates everything and tells
  you the one or two commands to run.
- **`n`: a module of the chenile-gen build**, with `jgen` as its parent, for blueprints that ship inside JGen.

Either way you get a blueprint that works straight away:

```text
bp-invoice/
  pom.xml
  src/main/resources/META-INF/blueprint.json             # inputs: projectName, projectVersion, includeExtras, ...
  src/main/resources/invoice-template/__projectName__/
      README.md.mustache                                 # a rendered file
      %%includeExtras=true%%/EXTRAS.md.mustache          # an optional folder
  src/main/java/org/chenile/jgen/blueprint/invoice/InitInvoiceBlueprint.java
  src/test/java/org/chenile/jgen/blueprint/invoice/InvoiceBlueprintTest.java
```

`mvn test` loads it the way JGen does and generates the starter project, so you can change one thing at a time
and keep a passing test.

## Step 2 — Add the template tree

The generated blueprint already has a template folder, `src/main/resources/invoice-template/`, with a starter
tree under `__projectName__/`. Replace it with the file and folder structure your blueprint should generate:

```text
src/main/resources/invoice-template/
  __projectName__/
    __projectName__-service/...
```

Use the [template conventions](/template-conventions/) — `.mustache` files, `__name__` path placeholders,
`%%key=value%%` conditional folders, `.filelist` multi-file outputs, `_DoT_` for dotfiles. The existing `bp-service` and
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

If the blueprint uses an added [extension](/extensions/) (say Pebble templates or a language pack's rule), list it in
`"requires": ["pebble", "terraform@1.2"]`. JGen then refuses the blueprint when the extension is missing, with a
clear message, instead of generating files that weren't rendered. The templates can be in any installed engine:
`.mustache`, `.peb`, or an added one.

## Step 4 — Write the init hook

> **Or write the hooks in JavaScript (2.5.1).** Declare `"scripts": {"postInputCapture": "scripts/hooks.js"}` in
> `blueprint.json` and put a `postInputCapture(inputs, jgen)` function in that file. Scripts run in a sandbox, need
> no build, and keep the blueprint template-only. See [Scripting hooks](/scripting/).

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
chain: `.mustache` contents, `__name__` paths, `_DoT_` for dotfiles (`_DoT_gitignore` becomes `.gitignore`),
`%%key=value%%` folders, and `.filelist` outputs. Nothing extra to enable.

## Step 6 — Make the blueprint available

Discovery is classpath-based: JGen sees every `META-INF/blueprint.json` on its classpath. There are two ways to
get yours there.

- **Drop the jar in a blueprints folder (no rebuild).** Run `mvn package` in the blueprint module and copy the
  jar to `~/.jgen/blueprints` (or the installation's `blueprints/` folder, or a folder listed in
  `JGEN_BLUEPRINTS`). An installed JGen picks it up on its next run. See
  [Adding blueprints without a rebuild](/install/#adding-blueprints-without-a-rebuild).
- **Template-only, straight from a repository (2.5.1).** If the blueprint has no Java init hook (JavaScript
  hooks are fine), it needs no jar:
  push a version tag and use it as `github:owner/repo@tag`, or try it locally as `dir:<folder>`. See
  [template-only blueprints](/remote-blueprints/#template-only-blueprints).
- **Publish it on GitHub.** With `standaloneRepo` = `y` this is already set up: push the local repository, then
  push a version tag and the generated workflow attaches the jar to a release. Anyone can then use it as
  `github:owner/repo@version`. See [Blueprints from the web](/remote-blueprints/).
- **Build it into JGen.** To ship a blueprint with JGen itself, add the module to the root `pom.xml` (under
  `<modules>`, and under `<dependencyManagement>` if you want it managed like the others) and add it as a
  dependency in `jgen-cli/pom.xml`.

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

## Generating a Git repository: `initGit`

<span class="pill">2.5.1</span>

Any blueprint can have JGen turn the project it generates into a local Git repository. No code is needed.

**Turning it on.** Put `initGit` = `true` in the input map. Either ask for it:

```json
{ "name": "initGit", "type": "BOOLEAN", "description": "Create a Git repository", "defaultValue": "y" }
```

or set it from the blueprint's `postInputCaptureHook`, from whatever conditions make sense:

```java
map.put("initGit", Boolean.TRUE.equals(map.get("standaloneRepo")));
```

**What JGen does.** With `initGit` true, after everything is generated and the generation record is written, JGen:

1. runs `git init` in the project folder, with branch `main`;
2. adds every file, including `.jgen/`;
3. makes one commit.

It never sets a commit author. If Git is not installed, or has no `user.name` and `user.email`, the project is still
generated, and JGen prints the exact commands to finish by hand. Only top-level generations are affected, never
child blueprints.

**Connecting it to GitHub.** To have the repository point at GitHub, declare an input field with the
`GITHUB_REPOSITORY` rule, and name it in `blueprint.json` with `gitRepositoryField`:

```json
{
  "name": "terraform-module",
  "templateFolder": "terraform-module-template",
  "gitRepositoryField": "githubRepository",
  "gitCommitMessage": "Initial ${projectName}, generated by JGen",
  "inputFields": [
    { "name": "projectName", "type": "STRING", "validation": "PROJECT_NAME" },
    { "name": "initGit", "type": "BOOLEAN", "defaultValue": "y" },
    { "name": "githubRepository", "type": "STRING", "required": false, "validation": "GITHUB_REPOSITORY",
      "visibleWhen": { "initGit": "true" } }
  ]
}
```

When the user gives a value, JGen adds it as the `origin` remote and prints the two steps that are left:

```text
Git repository ready: /…/output/infra (branch main, 1 commit)
Remote origin: git@github.com:acme/infra.git
Next:
  1. Create an empty repository on GitHub: acme/infra (no README, licence or .gitignore)
  2. git -C /…/output/infra push -u origin main
```

How the details work:

- **No repository given.** Without a value, or without a `gitRepositoryField`, you still get the local repository,
  plus a hint for adding a remote.
- **Remote style.** The remote uses SSH by default. Set `"gitRemoteProtocol": "https"` in your
  [config](/configuration/) for HTTPS remotes.
- **Load-time check.** `gitRepositoryField` must name an input field whose validation is `GITHUB_REPOSITORY`. This
  is checked when the blueprint is loaded.
- **What JGen never does.** It does not create the repository on GitHub and does not push. Both need your
  credentials.

**After the repository exists.** The blueprint's `finalHook` runs after this step. It can read `gitInitialised`,
`gitCommitted` and `gitRemote` from the map, for example to print project-specific next steps.

## Where the real logic lives

`jgen-base` defines the framework, `jgen-cli` handles interaction and input files, and the blueprint modules
define the actual output. In practice most user-visible changes are **template edits**, not Java changes. If you
want to change *global* generation behaviour, start in `jgen-base`; if a generated project is wrong for just one
blueprint, inspect that blueprint's templates first.
