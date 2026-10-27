---
title: "Why JGen? How it compares with other code generators"
eyebrow: "Getting started"
permalink: /why-jgen/
summary: "JGen generates Chenile projects better than anything else, because it was built for them. But the engine generates anything that is text. Here is how it compares with Cookiecutter, Copier, Yeoman, JHipster, Nx, Plop, Hygen and Backstage, and where each is the better choice."
---

Tailors have long worked from paper patterns. A pattern is not a garment. It is a precise, reusable description
of one: where to cut, where to fold, which parts are optional. Hand the same pattern to anyone with a pair of
scissors and you get the same garment every time, adjusted to the person wearing it.

A good code generator is a box of such patterns. The hard work is not producing files (any script can do that). It is
capturing the *pattern*: which choices a team is allowed to make, which values are valid, which parts appear only
when asked for, and how one pattern fits inside another. JGen was written to do that for Chenile, and it does it
very well. Along the way it became a general-purpose pattern box, which is the part I want to talk about here.

## What JGen is, in one paragraph

A JGen **blueprint** is one jar holding three things:

- a typed **input contract** (`blueprint.json`) that says what a user may choose and what counts as valid;
- a **template tree** that is copied, rendered with Mustache and reshaped by markup in its file and folder names;
- optional Java **hooks** that augment the input map, compute inputs for child blueprints, and touch the
  generated code afterwards.

The engine runs the same [pipeline](/pipeline/) for every blueprint. The CLI (`jgen -g` to emit a sample input,
`jgen -f` to generate from it), a web portal and other programs all drive that one engine. Fourteen blueprints
ship in the box, from a single Chenile service to a whole ecosystem of services, mini-monoliths and process
workers.

## It is not only for Chenile

Nothing in the engine knows what a Chenile service is. It copies a folder, renders text, renames paths, drops
or keeps conditional folders, splits multi-file templates and calls other blueprints. The Chenile knowledge lives
entirely in the Chenile blueprints. A blueprint for a Go service, a Terraform module, a React component library,
a set of Kubernetes manifests or a documentation site is written exactly the same way.

The honest qualifier: the shared project checks expect Java-style package segments (`com`, `company`, `org`)
and a project name. That is a small Chenile accent in the engine, and harmless for most projects. Everything
else is neutral.

## The things JGen does unusually well

### 1. The template tree does the heavy lifting

In Yeoman, Plop or Nx, a generator is a program that happens to write files. In JGen most of a blueprint's
behaviour is visible in its template tree, because file and folder **names** carry instructions:

- **Path placeholders.** `__service__/__Service__Application.java` becomes `orders/OrdersApplication.java`.
- **Conditional folders.** A folder named `%%jpa=true%%` is dissolved into its parent when the user chooses JPA,
  and dropped entirely when they don't. Persistence code, security wiring, a Dockerfile, a whole optional module:
  each lives in its own conditional folder, so the template shows at a glance what is optional and what decides
  it. There is no `if` buried in a hook, and no post-generation clean-up script deleting files that should never
  have been written.
- **File-list expansion.** A file ending in `.filelist.mustache` is rendered first and then split on
  `--START--<name>` / `--END--` markers into separate files. Because both the names and the bodies are rendered,
  one template can emit one class per entity, one listener per process type, one test per state, however many the
  input asks for. Most template-only generators can produce a fixed set of files; producing a *variable* number
  usually means writing code.
- **Child blueprint markers.** An empty folder named {% raw %}`%{{chenile-service forEach=services as=service}}`{% endraw %} runs
  another blueprint in place, once per element of a list. Put it inside a `%%flag=true%%` folder and it runs
  only when asked for.

The order is fixed and documented ([pipeline](/pipeline/)): Mustache first, then file lists, then path
placeholders, then conditional folders, then child blueprints. So the pieces combine predictably. Reviewing a
blueprint is mostly reading a folder; the [template reference](/template-conventions/) covers the whole
vocabulary.

### 2. Code exactly where templates run out

Templates are the right tool for most of a project and the wrong tool for some of it. JGen lets a blueprint
author add Java at four well-defined points, and nowhere else:

| Hook | When it runs | What it is for |
|---|---|---|
| `InitHook` | once, when the blueprint is loaded | register the other hooks; register sample-input emitters for file inputs |
| `postInputCaptureHook` | after inputs are validated, before any file is written | **augment the input map**: derive names (`orders` → `Orders`), or parse an uploaded file into structures the templates iterate over |
| `childBlueprintInputs` | for each child blueprint marker | **compute the child's input map** from the parent's inputs and the marker's attributes |
| `postProcessHook` | after generation | **touch the generated code**: copy a supplied file into place, relocate output, run a formatter or another tool |

The built-in blueprints show the range:

- **Workflow service.** It reads a Finito states XML in `postInputCaptureHook`, turning states and events into
  lists that drive file-list templates. Its `postProcessHook` then copies the original XML into the generated
  project.
- **Process management.** It reads a process-definition file and uses `childBlueprintInputs` to give each
  generated worker module its own inputs.

The hook receives the same map the templates see, so the division of labour stays simple. Code computes data;
templates turn data into files.

### 3. One jar per blueprint: template and code travel together

A blueprint is a single Maven module, built into a single jar. The jar holds the input contract
(`META-INF/blueprint.json`), the template tree and the hook classes; the module also holds the blueprint's own
tests. Nothing about the blueprint lives anywhere else.
Consequences I value more every year:

- **Modular by construction.** Adding a blueprint to JGen means adding a dependency. Removing it means removing
  the dependency. The registry finds every `blueprint.json` on the classpath; there is no central catalogue to
  edit and no plugin directory to keep in sync.
- **Versioned and published like any library.** A team can publish its blueprints to its own Maven repository,
  pin versions, and roll them forward deliberately. The template and the code that supports it can never drift
  apart, because they are released as one artefact.
- **Testable like any library.** A blueprint's tests sit in the same module as its templates and run in the normal build.
- **Shareable without a server.** A blueprint jar works the same in the CLI, the web portal, a CI job or another
  program.

npm-packaged Yeoman generators and Nx plugins share some of this. The difference is that a JGen jar is mostly
templates and data, with code only at the four hook points. Cookiecutter and Copier templates are git
repositories, which is convenient, but any supporting logic is hook scripts beside the template, not a tested,
versioned unit.

### 4. One validator, everywhere

Input fields declare their types and rules: `JAVA_IDENTIFIER`, `KAFKA_TOPIC`, `HTTP_URL`, dropdown values,
records with child fields, fields that are visible only when another field has a given value. The same validator
checks every input, whichever way it arrives:

- an interactive CLI session;
- a JSON input file;
- the web portal;
- a child blueprint called by its parent;
- another program, including an AI model proposing inputs.

It reports *every* problem at once, with a path such as `fields[2].name`, before a single file is written. See
[input validation](/input-validation/).

### 5. Composable output and version awareness

Every blueprint accepts an optional parent POM and extra dependencies, so generated projects slot under an
existing Maven reactor without template edits ([composable blueprints](/composable-blueprints/)). A blueprint
can also declare the minimum runtime version its output needs (`sinceVersion`), and JGen refuses to generate
code that will not run on the configured version.

## How it compares

The tools below are all good at what they set out to do. The table is about fit, not ranking.

| | Blueprint written as | Packaged as | Input validation | Generators composing generators | Updating generated projects | Runtime |
|---|---|---|---|---|---|---|
| **JGen** | typed contract + template tree with path markup (conditional folders, file lists) + four optional Java hooks | one versioned jar: contract, templates, hooks | declarative, all errors at once, same validator on every path | yes, marker folders (once, conditional, per list element) with computed child inputs | no | JVM |
| [Cookiecutter](https://github.com/cookiecutter/cookiecutter) | Jinja templates + `cookiecutter.json` + hook scripts | a git repo or folder | in hooks | no | no | Python |
| [Copier](https://copier.readthedocs.io/) | Jinja templates + `copier.yml` + tasks | a git repo with version tags | per-question validators | several templates can be applied to one project | **yes**, from tagged template versions | Python |
| [Yeoman](https://yeoman.io/) | JavaScript generator code + EJS | npm package | in code | yes, in code | no | Node |
| [JHipster](https://www.jhipster.tech/) | a full application generator (Yeoman-based) with JDL | npm packages | strong, for its own domain model | blueprints override parts of it | yes, `jhipster upgrade` | Node |
| [Nx generators](https://nx.dev/) | TypeScript against a file-tree API + JSON schema | npm plugin | JSON schema | yes, in code | yes, migrations | Node, Nx workspace |
| [Plop](https://plopjs.com/) / [Hygen](https://www.hygen.io/) | Handlebars / EJS snippets + JS prompts | files in your repo | in code | no | no | Node |
| [Backstage Scaffolder](https://backstage.io/docs/features/software-templates/) | Nunjucks templates + actions | a template entity in a developer portal | form schema | step pipelines | no | a Backstage installation |
| Spring Initializr / Maven archetypes | fixed project skeletons | a service / an archetype jar | minimal | no | no | JVM |

A few notes, in fairness to the others:

- **Copier keeps projects up to date.** `copier update` replays newer template versions onto an existing project.
  JGen generates once. If keeping dozens of repositories in step with their template is your main problem, Copier
  is the stronger choice today.
- **JHipster is a product, JGen is a kit.** JHipster generates a complete, opinionated application stack and
  upgrades it; it is the right answer if its stack is your stack. JGen gives you the means to capture *your*
  stack.
- **Nx is excellent inside an Nx monorepo.** Its generators and migrations are tightly integrated with the
  workspace. JGen does not assume any repository layout.
- **Plop and Hygen are for small, frequent snippets** inside an existing codebase: a component here, a reducer
  there. JGen is happiest generating whole projects and modules.
- **Backstage is an organisational front door**, with access control and catalogue integration. JGen can sit
  behind something like it as the engine that produces the code.
- **Initializr and archetypes are the simplest way to get one Spring or Maven skeleton.** They stop there.

## Where JGen is not the right tool (yet)

- It does not update projects it has already generated.
- The flip side of one-jar packaging: writing a blueprint means a small Maven build, even if the output is pure
  YAML, and hooks are written in Java.
- The community is small compared with Cookiecutter's or Yeoman's, so you will be writing more of your own
  blueprints than downloading them.

## Why it suits AI-assisted development

A typed input contract plus a validator that lists every error is exactly what a language model needs. The model
proposes a JSON input, JGen says precisely what is wrong, the model fixes it, and only a valid input ever reaches
generation. The model never writes the project's code directly; it fills in a pattern that a person designed and
reviewed. This is how ChenAIle, the AI assistant we are building on top of Chenile, drives JGen, and it works the same way
for any blueprint.

## Try it

```bash
git clone https://github.com/rajakolluru/chenile-gen.git && cd chenile-gen
make clean all && source setpath.sh          # puts jgen on PATH
jgen -g chenile-service -o orders.json       # emit a sample input
jgen -f orders.json                          # generate ./output/orders
```

Then emit the input for a new blueprint with `jgen -g jgen-blueprint`, generate it, and follow
[Write your own blueprint](/authoring/). If you build a blueprint for something that is not Chenile at all, I
would love to hear about it.
