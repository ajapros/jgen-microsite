---
title: "Install & the CLI"
eyebrow: "Getting started"
permalink: /install/
summary: "Install JGen with Homebrew, JBang or a zip, add blueprint jars without a rebuild, and learn the CLI flags and flows."
---

## Install

JGen needs Java 25 or later. Pick one of these; none of them needs a build.

**Homebrew (macOS, Linux)**

```bash
brew install ajapros/jgen-tap/jgen
```

Homebrew also installs OpenJDK if you don't have it.

**JBang (macOS, Linux, Windows)**

```bash
jbang app install jgen@rajakolluru/chenile-gen
```

[JBang](https://www.jbang.dev/) downloads a Java runtime if it needs one, puts `jgen` on your PATH and always fetches
the latest release. Run `jbang app install --force jgen@rajakolluru/chenile-gen` to upgrade.

**Download the zip**

Download `jgen-<version>.zip` from the
[GitHub releases](https://github.com/rajakolluru/chenile-gen/releases), unzip it anywhere and add its `bin`
folder to your PATH. It contains `bin/jgen` (and `bin/jgen.cmd` for Windows), `lib/jgen-cli.jar` and an empty
`blueprints/` folder.

Check the installation with `jgen --version`.

**Tab completion.** Homebrew sets up completion for bash, zsh and fish. With JBang or the zip, enable it once:

```bash
echo 'source <(jgen --completion bash)' >> ~/.bashrc                    # bash
echo 'source <(jgen --completion zsh)'  >> ~/.zshrc                     # zsh (after compinit)
jgen --completion fish > ~/.config/fish/completions/jgen.fish           # fish
```

Tab then completes options, blueprint names after `-g` (installed blueprints, cached `github:` references and
the `github:` prefix), JSON files after `-f` and `-c`, and folders after `-r` and `--into`.

## Adding blueprints without a rebuild

Every blueprint is a single jar, and JGen picks up blueprint jars from these folders at start-up:

| Folder | Use |
| --- | --- |
| `~/.jgen/blueprints` | your own blueprints, for every JGen installation (also works with JBang) |
| `<install>/blueprints` | blueprints shared by everyone using that installation (zip installs) |
| `$JGEN_BLUEPRINTS` | extra folders, comma-separated, e.g. a team share or a CI cache |

```bash
mkdir -p ~/.jgen/blueprints
cp target/bp-terraform-module-1.0.jar ~/.jgen/blueprints/
jgen -g terraform-module            # available immediately
```

Use the plain jar that `mvn package` builds for the blueprint module. Don't shade JGen's own libraries into it.
You can also skip the copying: `jgen -g github:owner/repo` fetches a blueprint from its GitHub release (see
[Blueprints from the web](/remote-blueprints/)).
See [Write your own blueprint](/authoring/) for building one.

## Build from source

Use this when you are working on JGen itself:

```bash
git clone https://github.com/rajakolluru/chenile-gen.git && cd chenile-gen
make clean all && source setpath.sh          # builds the modules, puts jgen.sh and stm-cli on PATH
```

The build also produces the distribution zip, `jgen/jgen-cli/target/jgen-cli-<version>-dist.zip`, which is
what a release publishes. `jgen.sh` runs the latest `jgen-cli-*.jar` under `target/`.

Generated projects build with `mvn install` (or `make build`); generated mini-monoliths run with `make run`
(port 8080 by default).

## CLI options

The complete reference is the [jgen(1) manual](/manual/), also installed as `man jgen`.

| Flag | Meaning |
| --- | --- |
| `-c, --config` | Load a config JSON instead of the bundled default config. |
| `-e, --emit-config` | Copy global config into `config/`; with `-g`, also emit companion sample input files. |
| `-f, --input-file` | Read blueprint inputs from a JSON file (non-interactive). |
| `-g, --generate-blueprint-file` | Emit a sample input JSON for a named blueprint. |
| `-o, --output-file` | Where to write the sample input JSON. |
| `-r, --regenerate` | Re-cut a generated project with the current blueprint into a separate folder and classify every file (2.1.35; see [Regeneration](/regeneration/)). |
| `--into` | With `-r`, the folder to regenerate into (default `<project>.regenerated`). |
| `--blueprint` | With `-r`, regenerate with this blueprint (e.g. a newer `github:` reference). |
| `--fetch` | Fetch, verify and cache a blueprint from GitHub and describe it ([Blueprints from the web](/remote-blueprints/)). |
| `--trust` | Trust the blueprint sources used in this run without asking. |
| `--offline` | Use only installed and cached blueprints. |
| `--completion` | Print the completion script for `bash`, `zsh` or `fish`. |

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
