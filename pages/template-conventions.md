---
title: "Template reference"
eyebrow: "Authoring blueprints"
permalink: /template-conventions/
summary: "Every piece of special markup a blueprint template understands — Mustache expressions in file contents, and the path, conditional, file-list and aggregation markup in file and folder names."
---

The template folder under each `bp-*` module is where most of a blueprint's behaviour lives. JGen gives every
blueprint the same markup, applied by the shared [processor chain](/pipeline/) in a fixed order. There are two
families of markup: **Mustache expressions inside file contents**, and **special markers in file and folder
names**. This page documents all of them.

## The data available to every expression

All markup resolves against one `Map<String,Object>` — the *template map* — assembled before rendering from, in
order:

1. **Config defaults** — `com`, `company`, `org`, `chenilePackage`, `chenileVersion`, default service/version/
   destination, and anything else in the selected [config file](/configuration/). `${…}` references in field
   defaults are resolved here.
2. **Your blueprint inputs** — every field the user supplied (booleans arrive as the string `"true"` or
   `"false"`; file/record inputs arrive as parsed structures).
3. **Derived values from the init hook** — whatever the blueprint's `postInputCaptureHook` adds, such as a
   capitalized `Service` from `service`.
4. **Lambdas** — helper functions the engine registers (see [capitalize](#the-capitalize-lambda)).

A Mustache expression and a `__placeholder__` in a path read from this same map — they only differ in **where**
they appear and **when** they run.

---

# Markup in file contents (Mustache)

Files ending in `.mustache` are rendered with [Mustache](https://github.com/spullara/mustache.java) (the
`mustache.java` engine) and written **without** the `.mustache` suffix. If the rendered file ends in `.sh`, JGen
marks it executable.

## Variables — escaped vs. unescaped

```mustache
{% raw %}{{service}}      → HTML-escaped substitution
{{{service}}}    → raw, unescaped substitution
{{&service}}     → raw, unescaped substitution (alternate form){% endraw %}
```

This distinction matters. Mustache **HTML-escapes** `{% raw %}{{x}}{% endraw %}` by default, so `&` becomes
`&amp;`, `<` becomes `&lt;`, and `"` becomes `&quot;`. That is wrong for the Java, JSON, YAML, `.properties` and
shell files a blueprint usually generates. **Use the triple-stache `{% raw %}{{{x}}}{% endraw %}` (or
`{% raw %}{{&x}}{% endraw %}`) whenever the value may contain a character that HTML-escaping would corrupt** —
for example a JSON string, a URL with query parameters, or anything with `&`, `<`, `>` or quotes. Built-in
blueprints use the triple form in their `defs.json`, `application.properties` and listener templates for exactly
this reason.

## Sections, inverted sections and lists

```mustache
{% raw %}{{#jpa}} ...rendered when jpa is truthy... {{/jpa}}
{{^jpa}} ...rendered when jpa is falsy/absent... {{/jpa}}
{{#dependencies}}
  <artifactId>{{artifactId}}</artifactId>   {{! iterated once per list element }}
{{/dependencies}}{% endraw %}
```

A section `{% raw %}{{#key}}{% endraw %}` renders its body when the key is truthy (a non-empty list renders once
per element, with the element's fields in scope); an inverted section `{% raw %}{{^key}}{% endraw %}` renders
when the key is absent, false, or an empty list. Because enabled booleans are the string `"true"`, a
`{% raw %}{{#jpa}}{% endraw %}` section is the content-file equivalent of a `%%jpa=true%%` folder.

## The `capitalize` lambda

JGen registers one template lambda, `capitalize`, which upper-cases the first character of the text it wraps.
Use it as a section around the value to transform:

```mustache
{% raw %}{{#capitalize}}{{processType}}{{/capitalize}}Executor{% endraw %}
```

With `processType = "record"` this renders `RecordExecutor`. (The engine's `CapUtils` also offers
`capitalizeHyphens` for joining hyphenated names into CamelCase, but that is used by Java init hooks, not
exposed as a template lambda — compute such names in the hook and put the result in the map.)

## Comments, partials and delimiters

Standard `mustache.java` features are available: comments `{% raw %}{{! like this }}{% endraw %}`, partials
`{% raw %}{{> partial }}{% endraw %}`, and set-delimiter changes `{% raw %}{{=<% %>=}}{% endraw %}`. Comments are
handy for annotating a template without affecting output.

---

# Markup in file and folder names

The following markers appear in **names**, not contents, and are applied by dedicated pipeline steps *after*
Mustache rendering. (A name-marker folder is skipped by the path replacer so it survives until its own step.)

## `__key__` — path placeholders

A file or directory name segment of the form `__key__` is replaced with the value of `key` in the template map.
Replacement happens **after** file contents are rendered.

```text
__service__/__service__-service        →  orders/orders-service
__Service__Application.java            →  OrdersApplication.java
```

Common keys: `__service__`, `__Service__`, `__application__`, `__workerModule__`, `__monolith__`, and the
package segments `__com__` / `__company__` / `__org__`. Capitalized forms such as `__Service__` are values the
init hook derives — the placeholder itself is just another map lookup.

Path substitution is backed by Apache Commons Text `StringSubstitutor` with the `__` prefix/suffix, so a
**default value** can be supplied inline when a key might be missing:

```text
__port:-8080__          →  the value of port, or 8080 if port is unset
```

Values are looked up from the same template map, and a replaced value that itself contains `__…__` is resolved
recursively.

## `%%key=value%%` — conditional folders

A folder named `%%key=value%%` is **conditional**. If the template map's value for `key`, as a string, equals
`value`, the folder's contents are promoted into its parent; otherwise they are dropped. **Either way the
`%%…%%` folder itself is removed.**

```text
%%jpa=true%%/           →  contents promoted when jpa == "true", else discarded
%%security=true%%/
```

Exact semantics to keep in mind:

- The split is on the **first** `=`; everything after it is the literal expected value.
- The comparison is plain string equality against the map value's `toString()` — there is no expression
  language, no negation, and no `!=`. For the opposite case, use a second folder with the other value, or an
  inverted Mustache section inside a file.
- Enabled booleans are the string `"true"`, so the idiom is `%%flag=true%%`.
- Conditional folders run **before** aggregation markers expand, so wrapping a `%{{…}}` marker in a
  `%%flag=true%%` folder is how you include a child blueprint only when a flag is set — see
  [Aggregated blueprints](/aggregated-blueprints/).

## `.filelist` — one template, many files

A file ending in `.filelist` is a container that the engine splits into several files, each written to the same
folder. The format is a sequence of `--START--<path>` / `--END--` blocks:

```text
--START--RecordExecutor.java
...contents of RecordExecutor.java...
--END--
--START--AuditExecutor.java
...contents of AuditExecutor.java...
--END--
```

The text on the `--START--` line after the marker is the output filename; the lines up to `--END--` are its
contents. The source `.filelist` is deleted after splitting.

**The powerful combination is `*.filelist.mustache`.** Because Mustache rendering (step 7) runs **before** the
file-splitter (step 8), a filelist can be a Mustache template *first* and a multi-file container *second* — so
both the `--START--` filenames **and** the file bodies can be computed from the map. This is how one template
emits a variable number of files — for example one worker class per element of a list:

```mustache
{% raw %}{{#processes}}
--START--{{#capitalize}}{{name}}{{/capitalize}}Executor.java
package {{com}}.{{company}}.{{org}}.{{application}}.worker;
public class {{#capitalize}}{{name}}{{/capitalize}}Executor
        extends ExecutorWorkListener<{{inputType}}, {{outputType}}> { }
--END--
{{/processes}}{% endraw %}
```

With three processes in the list, that single source renders and then splits into three `…Executor.java` files.

## `%{{blueprint attr=value}}` — aggregation markers

An **empty** folder named `%{{blueprint …}}` invokes another registered blueprint at that location — once,
conditionally, or once per list element. The marker's grammar is strict:

```text
%{{chenile-service name=orders}}
%{{minimonolith role=worker label="Background workers"}}
%{{chenile-process-workers forEach=processes as=process}}
```

- Prefix `%{{`, suffix `}}`.
- The **first token** is the exact registered blueprint name, matching `[A-Za-z0-9][A-Za-z0-9_.-]*`.
- Then zero or more `key=value` **attributes**. A key matches `[A-Za-z_][A-Za-z0-9_.-]*`; no spaces around `=`.
- A value is either **unquoted** (no whitespace, and no `'`, `"` or `=` inside) or **quoted** with single or
  double quotes. Inside a quoted value, `\` escapes only the matching quote or a backslash. An **empty value
  must be quoted** (`label=''`).
- Duplicate keys, a missing value, an unterminated quote, a malformed name, or text after a closing quote with
  no whitespace are all rejected with an identifying error.
- `forEach` and `as` are **reserved** expansion attributes for repeating a child over a list.

The marker folder must be empty (an empty `.gitkeep` is allowed so Git keeps it). Attributes are passed to the
aggregate's `childBlueprintInputs` callback as strings — they are **not** auto-merged into the child's inputs.
The full lifecycle, the callback, conditional inclusion and list expansion are covered in
[Aggregated blueprints](/aggregated-blueprints/).

---

## When each marker is applied

Order matters because each marker sees the tree as the previous step left it. From the
[generation pipeline](/pipeline/):

| Step | Marker | Acts on |
| --- | --- | --- |
| 7 `file-template-executor` | `{% raw %}{{ }}{% endraw %}`, `{% raw %}{{{ }}}{% endraw %}`, sections, `capitalize` | **contents** of `.mustache` files |
| 8 `file-splitter` | `--START--` / `--END--` | `.filelist` files (already rendered) |
| 9 `file-name-replacer` | `__key__` | file and folder **names** |
| 10 `conditional-processor` | `%%key=value%%` | folder **names** |
| 11 `blueprint-expander` | `%{{blueprint …}}` | empty marker **folders** |

The practical consequences: Mustache in a file is resolved before that file's path placeholders; a
`.filelist.mustache` is rendered before it is split, so its filenames can be dynamic; and conditional folders
are selected before child blueprints expand, so a marker inside an excluded `%%…%%` folder is never invoked.

## Quick rules

- Generating Java, JSON, YAML, `.properties` or shell? Reach for `{% raw %}{{{ }}}{% endraw %}` unless you
  genuinely want HTML escaping.
- Need a capitalized identifier? `{% raw %}{{#capitalize}}{{x}}{{/capitalize}}{% endraw %}` in a file, or
  compute it in the init hook for use in a `__Name__` path.
- Need an optional feature's files? A `%%flag=true%%` folder for whole subtrees, an inverted section
  `{% raw %}{{^flag}}{% endraw %}` for a few lines.
- Need a variable number of files? A `*.filelist.mustache` driven by a list.
- Need to compose other blueprints? An empty `%{{blueprint …}}` folder plus a `childBlueprintInputs` callback.
