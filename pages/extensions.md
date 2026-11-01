---
title: "Extending JGen"
eyebrow: "Authoring blueprints"
permalink: /extensions/
summary: "Template engines, language packs and path tags: add a template language, support another target language, or add a new tag, without changing JGen. Blueprints declare what they need with requires."
---

<span class="pill">2.1.36</span>

JGen started as a generator for Chenile's Java services, but its engine doesn't care what it generates. Three
**extension points** let you take it elsewhere without forking it:

| Kind | What it adds | Built in |
| --- | --- | --- |
| **Template engine** | a template language, chosen by file extension | `mustache` (`.mustache`); `pebble` (`.peb`) ships with JGen |
| **Language pack** | validation rules and template helpers for a target language | `python`, `node` |
| **Path tag** | a rule for file and folder names | `filelist`, `placeholder`, `dot`, `conditional` |

Unlike [hooks](/scripting/), which belong to one blueprint, extensions are installed once and every blueprint can
use them. `jgen --extensions` lists what is installed.

## Template engines

The render step picks an engine by the template's extension and drops the extension:

```text
src/__module__.py.peb        → rendered by pebble   → src/orders.py
pom.xml.mustache             → rendered by mustache → pom.xml
README.md                    → copied as it is
```

A blueprint can mix engines freely. Mustache stays the default for Java because it is logic-less. Pebble (Twig and
Jinja syntax) suits languages where you want real conditionals, loops with filters, and whitespace control:

{% raw %}
```jinja
class {{ entity | pascalCase }}:
{% for field in fields %}    {{ field.name | snakeCase }}: {{ field.type }}
{% endfor %}
```
{% endraw %}

JGen's Pebble is set up for code, not HTML: output isn't escaped, a missing value renders as empty (as in
Mustache), and every [template helper](#template-helpers) is a filter. Templates are self-contained: `include` and
`extends` aren't supported.

## Language packs

A pack teaches JGen about a target language. Today that means two things.

**Validation rules.** Blueprints name them in `validation` exactly like the built-in rules:

```json
{ "name": "module",  "type": "STRING", "validation": "PYTHON_MODULE" },
{ "name": "package", "type": "STRING", "validation": "NPM_PACKAGE" }
```

| Pack | Rule | Accepts |
| --- | --- | --- |
| `python` | `PYTHON_MODULE` | `order_service`: lowercase letters, digits and `_`, not a keyword |
| | `PYTHON_PACKAGE` | `acme.orders`: dot-separated module names |
| | `PYPI_PROJECT` | `acme-orders`: a PyPI project name |
| `node` | `NPM_PACKAGE` | `@acme/orders`: lowercase, optionally scoped, at most 214 characters |
| | `JS_IDENTIFIER` | `orderService`: an identifier that is not a reserved word |

A rule that no installed pack defines is an error as soon as the field is read: when the blueprint loads, not
when someone first uses it. JGen checks a blueprint's [`requires`](#requires-what-a-blueprint-needs) first, so a
blueprint that needs a missing pack is told which pack is missing, rather than that its rule is unknown.

**Template helpers.** A pack can add helpers next to the core ones. They reach every engine and
[script](/scripting/).

### Template helpers

| Helper | `purchase order` becomes |
| --- | --- |
| `capitalize` / `uncapitalize` | first letter changed only |
| `camelCase` | `purchaseOrder` |
| `pascalCase` | `PurchaseOrder` |
| `snakeCase` | `purchase_order` |
| `upperSnakeCase` | `PURCHASE_ORDER` |
| `kebabCase` | `purchase-order` |

Words are split at spaces, `-`, `_` and case changes, so `purchaseOrder`, `PurchaseOrder` and `purchase-order` all
give the same results, and `HTTPServer` becomes `http_server`.

{% raw %}
| Where | How |
| --- | --- |
| Mustache | `{{#snakeCase}}{{entity}}{{/snakeCase}}` |
| Pebble | <code>{{ entity &#124; snakeCase }}</code> |
| JavaScript hooks | `jgen.snakeCase(inputs.entity)`, or `jgen.helper("anyHelper", text)` |
{% endraw %}

## Path tags

Path tags are covered in the [template reference](/template-conventions/#path-tags-and-adding-your-own). Each tag
has an `order`, which sets when it runs relative to the others.

## Writing an extension

An extension is an ordinary jar. Implement one of these interfaces from `jgen-base` (a `provided` dependency):

```java
public interface TemplateEngine {                      // org.chenile.jgen.ext
    String name();                                     // "freemarker"
    String extension();                                // ".ftl"
    void render(Reader template, String templateName, Map<String,Object> data, Writer out) throws Exception;
}

public interface LanguagePack {                        // org.chenile.jgen.ext
    String name();                                     // "terraform"
    default List<ValidationRule> rules() { ... }       // ValidationRule.of("TF_RESOURCE", "a resource name", "[a-z][a-z0-9_]*", null)
    default Map<String, Function<String,String>> helpers() { ... }
}

public interface PathTag { ... }                       // org.chenile.jgen.template.tags
```

List each class in the matching services file:

```text
META-INF/services/org.chenile.jgen.ext.TemplateEngine
META-INF/services/org.chenile.jgen.ext.LanguagePack
META-INF/services/org.chenile.jgen.template.tags.PathTag
```

Then drop the jar into a blueprints folder: `~/.jgen/blueprints`, the installation's `blueprints/`, or a folder in
`JGEN_BLUEPRINTS`. Add `addDefaultImplementationEntries` to the jar's manifest so that blueprints can ask for a
minimum version. The `jgen-ext-pebble` module in chenile-gen is a complete example.

**Rules that keep extensions safe:**

- **Unique names.** Extension names are unique across all three kinds, so a built-in can't be replaced.
- **Unique claims.** No two engines may claim the same file extension.
- **No overriding built-ins.** A pack can't redefine a built-in rule such as `JAVA_IDENTIFIER`, or an existing
  helper.

## `requires`: what a blueprint needs

A blueprint that depends on an extension says so in `blueprint.json`:

```json
{
  "name": "fastapi-service",
  "templateFolder": "fastapi-template",
  "requires": ["pebble", "python", "terraform@1.2"],
  "inputFields": [ ... ]
}
```

- **Format.** Each entry is a name, optionally with a minimum version (`name@version`, compared with the extension
  jar's `Implementation-Version`). The `kind:name` form is accepted too.
- **Missing extensions.** A blueprint whose requirements aren't installed is refused when it loads, before its
  input fields are read. The message
  names what is missing and where to put the jar.
- **The generation record.** The [generation record](/regeneration/) lists the added (not built-in) extensions the
  project depended on, as `kind:name@version`:
  - every added path tag;
  - the added engines that rendered files;
  - the packs whose rules the blueprint uses;
  - whatever it requires.

  `jgen -r` refuses to regenerate a project when one of them is missing, rather than produce different output.
