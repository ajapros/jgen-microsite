---
title: "Declarative input validation"
eyebrow: "Authoring blueprints"
permalink: /input-validation/
summary: "The reusable validation rules a blueprint field can declare, where they run, and how they apply to nested records and multi-select inputs."
---

Blueprints select reusable validation rules in `META-INF/blueprint.json`. The optional `validation` property
belongs to an input field (including a child field):

```json
{
  "name": "serviceName",
  "type": "STRING",
  "validation": "JAVA_IDENTIFIER",
  "description": "Java identifier for the service"
}
```

The framework checks the rule during interactive CLI capture, input-file and portal submission, and **again**
before blueprint hooks or any generated file write (the `generic-input-validator` step). The execution check
also covers inputs supplied by an [aggregated blueprint](/aggregated-blueprints/). Failures identify the field
and rule, with indexed paths for nested records.

## Built-in rules

| Rule | Accepted values |
| --- | --- |
| `JAVA_IDENTIFIER` | An ASCII letter followed by letters, digits or underscores; Java keywords are rejected. |
| `JAVA_TYPE` | A `JAVA_IDENTIFIER` starting with an uppercase letter. |
| `JAVA_PACKAGE` | Dot-separated `JAVA_IDENTIFIER` segments; no empty segments. |
| `PROJECT_NAME` | A letter followed by letters, digits, underscores or hyphens. |
| `VERSION` | Letters, digits, dots, underscores and hyphens. |
| `HTTP_URL` | An absolute HTTP or HTTPS URL with a host. |
| `KAFKA_TOPIC` | 1–249 ASCII letters, digits, dots, underscores or hyphens; excludes `.` and `..`. |
| `GITHUB_REPOSITORY` (2.1.36) | `owner/repo`, e.g. `acme/jgen-terraform`. The owner has at most 39 letters, digits or single hyphens and does not start or end with a hyphen. The repository has 1–100 letters, digits, dots, underscores or hyphens, and is not `.` or `..`. |
| `PATTERN` (2.1.36) | Matches the field's own `pattern` (see [below](#your-own-format-pattern)). |

A blueprint's `gitRepositoryField` must name a field with the `GITHUB_REPOSITORY` rule. That field's value becomes
the generated repository's `origin` (see [Generating a Git repository](/authoring/#generating-a-git-repository-initgit)).

Rule names are case-sensitive, and an unknown rule is a blueprint configuration error. Leaving `validation` out
preserves the existing field-type validation. Destination checks and the `sinceVersion` runtime check still run
independently. `required`, defaults, dropdown `validValues` and `visibleWhen` keep their meaning: hidden fields
are skipped, absent optional values are accepted, and resolved values are validated. **Validation never rewrites
a name to make it valid** — an invalid value fails rather than being silently corrected.

## Rules from language packs

<span class="pill">2.1.36</span>

Language packs add rules for other target languages. Blueprints name them the same way, for example
`"validation": "PYTHON_MODULE"`. The `python` and `node` packs are built in:

| Rule | Accepted values |
| --- | --- |
| `PYTHON_MODULE` | Lowercase letters, digits and `_`, not starting with a digit, not a Python keyword: `order_service`. |
| `PYTHON_PACKAGE` | Dot-separated `PYTHON_MODULE` names: `acme.orders`. |
| `PYPI_PROJECT` | Letters, digits, `.`, `_` or `-`, starting and ending with a letter or digit: `acme-orders`. |
| `NPM_PACKAGE` | Lowercase, optionally scoped, at most 214 characters: `@acme/orders`. |
| `JS_IDENTIFIER` | A JavaScript identifier that is not a reserved word: `orderService`. |

A rule name that is neither built in nor defined by an installed pack is rejected as soon as the field is read,
so a typo such as `"validation": "PYTHON_MODUEL"` fails the blueprint when it loads. A blueprint whose rules come from an added
pack should list that pack in `requires`. See [Extending JGen](/extensions/#language-packs).

## Your own format: `PATTERN`

<span class="pill">2.1.36</span>

When no named rule fits a format that is specific to your blueprint, use `PATTERN` with a regular expression and
a message:

```json
{
  "name": "ticket",
  "type": "STRING",
  "validation": "PATTERN",
  "pattern": "[A-Z]{2,5}-[0-9]+",
  "patternMessage": "must look like ABC-123"
}
```

- The **whole** value must match. Don't add `^` and `$`; a pattern that matches only part of the value fails.
- `patternMessage` is required and is the error users (and AI models) see: `Field ticket must look like ABC-123`.
  Write it as what a valid value looks like.
- The pattern is checked when the blueprint is loaded. A missing pattern, a missing message, an invalid regular
  expression, or a `pattern` on a field whose rule isn't `PATTERN` is a blueprint error, reported before anyone
  uses it.
- On a `MULTI_SELECT` field, each selected value must match.
- Patterns use Java regular-expression syntax (`java.util.regex`).

Prefer a named rule whenever one exists. If several blueprints need the same format, it should become a named
rule in `jgen-base`, so the format and its message stay the same everywhere.

## Rules on nested records and multi-selects

For a `RECORD_ARRAY`, put the rules on the child fields; record children are validated recursively:

```json
{
  "name": "fields",
  "type": "RECORD_ARRAY",
  "required": false,
  "childFields": [
    { "name": "name", "type": "STRING", "validation": "JAVA_IDENTIFIER" }
  ]
}
```

A rule on a multi-select input applies to each selected value. Structured Maven inputs keep their `SuperPOM` and
`POM_Dependencies` normalization and coordinate validation (see [Composable blueprints](/composable-blueprints/)).

## Validating a whole payload from code

`BlueprintPayloadValidator` (in `jgen-base`, 2.1.35) is the one validator every input path uses: `jgen -f`,
the portal, composed child blueprints through `BlueprintInputService.buildInputMap`, and external callers such
as ChenAIle. It resolves a payload as generation will (config defaults, `${...}` substitution, `visibleWhen`,
boolean and Maven-coordinate normalization) and reports **every** problem, not just the first.

```java
PayloadValidation result = BlueprintPayloadValidator.validate(blueprint, config, json); // config may be null
if (!result.valid()) result.errors().forEach(System.err::println);   // e.g. "fields[0].name: Field fields[0].name must satisfy JAVA_IDENTIFIER"
Map<String, Object> inputs = result.orThrow();                       // resolved inputs, or an exception listing all errors
```

Each `PayloadIssue` has a `path` (`$`, a field name, or a nested path such as `fields[0].name`), a `code` and a
`message`:

| Code | Kind | Meaning |
| --- | --- | --- |
| `INVALID_JSON` | error | Not valid JSON, empty, or a key appears twice (with line and column). |
| `NOT_AN_OBJECT` | error | Valid JSON, but not an object of field values. |
| `REQUIRED` | error | A required field that is visible is missing or empty. |
| `INVALID_VALUE` | error | Wrong type or shape, a value outside `validValues`, or a broken `validation` rule. Booleans must be `true`/`false`/`y`/`n`; other text is no longer silently read as false. |
| `INVALID_INPUT` | error | A generic input every blueprint shares: `com`/`company`/`org` segments, the project name, a `*Version`, or the registry URL. |
| `UNKNOWN_FIELD` | warning | A key the blueprint does not declare; generation ignores it. The `blueprint` key of an input file is exempt. |
| `NOT_VISIBLE` | warning | A field supplied while its `visibleWhen` condition is false; generation ignores it. |

The validator never touches the filesystem. Two checks stay at the generation boundary: the destination folder
must not already exist, and a `FILE` field must name an existing file. `validateFields(...)` runs the field
checks without the generic ones; `buildInputMap` uses it because the generic checks run on the final, composed
map before generation. The older fail-fast helpers (`InputValidation.validateBlueprint` and `validateField`,
`BlueprintValidationUtils.validateGenericInputs`) are now built on the same collect-all checks and throw the
first problem.

With `jgen -f`, the CLI runs the validator on the input file before generating, prints warnings, and stops with
the full list of errors.

## Where validation belongs

Put common scalar validation in these declarations. Leave relationship and domain-specific checks — process
hierarchy, cron payload contracts, and the like — to the blueprint's init hook. To add a new shared rule, extend
`ValidationType`, `InputValidation.validate` and `InputValidation.describe` in `jgen-base` and document its
contract. For a one-off format, use `PATTERN`.

## How the built-ins declare rules

Service, headless-service, batch, interceptor, integration-test application, query-namespace, mini-monolith and
generated-blueprint names use `JAVA_IDENTIFIER` — their templates insert those names directly into Java
packages. Ecosystem child service/namespace/monolith names use the same rule, so an invalid child name fails
before ecosystem generation starts. The integration-test entity uses `PROJECT_NAME` to keep hyphenated names in
routes and feature filenames. Workflow service names keep `PROJECT_NAME`, as do the ecosystem outer project and
the process worker JAR name (these are Maven project names). Versions, URLs, Kafka topics and package arrays use
their matching rules; free-text instructions, runtime-property values, file paths, booleans and dropdowns keep
their field-type checks.
