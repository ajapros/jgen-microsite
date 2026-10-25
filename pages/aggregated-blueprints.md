---
title: "Aggregated blueprints"
eyebrow: "Authoring blueprints"
permalink: /aggregated-blueprints/
summary: "A blueprint can invoke other blueprints through marker folders — once, conditionally, or once per list element — to compose whole multi-module applications."
---

An aggregated blueprint includes other registered blueprints using special placeholder folders in its template
tree. The enclosing blueprint computes each child's inputs through a callback; the child then runs its usual
defaults, validation, hooks and generation pipeline under the same JGen config. `chenile-ecosystem`,
`chenile-process-management` and `chenile-process-worker-set` are the built-in aggregates.

## Marker syntax

```text
%{{chenile-service name=orders}}
%{{minimonolith role=server}}
%{{minimonolith role=worker label="Background workers"}}
```

The first token is the exact registered blueprint name (not its Maven module name). The remaining `key=value`
tokens are passed to the callback as a `Map<String,String>` — values stay strings, including `true` and numbers,
and are **not** auto-merged into the child's input map. Use single or double quotes for values with spaces or
`=`; an empty value must be quoted (`label=''`); inside a quoted value, backslash escapes the matching quote or a
backslash. Attribute names start with a letter or underscore and may contain letters, digits, underscores, dots
and hyphens; no spaces around `=`. Duplicate keys, unterminated quotes, missing values and malformed markers are
rejected.

Markers are literal folder names, so they must obey the host filesystem's rules. Attributes are not Mustache
expressions or filename substitutions — pass complex values and paths through the input map, not through marker
attributes.

## Template layout and output placement

```text
aggregate-template/
  __application__/
    pom.xml.mustache
    %{{chenile-service name=orders}}/     .gitkeep
    %{{minimonolith role=server}}/        .gitkeep
    %{{minimonolith role=worker}}/        .gitkeep
```

Marker folders must be **empty** (an empty `.gitkeep` is allowed so Git retains them; any other contents are
rejected). At generation the marker disappears and its parent directory receives the child's normal output tree:

```text
example/
  pom.xml
  orders/     orders-api/  orders-service/
  server/     server-configurations/  server-package/
  worker/     worker-configurations/  worker-package/
```

JGen controls the child's effective `destFolder` from the marker's location; a value returned by the callback
cannot redirect output elsewhere. The **aggregate owns its reactor POM, module list and dependency
relationships** — JGen does not infer them. Supply `super_pom` and the child's declared `POM_Dependencies` using
the [composable input contract](/composable-blueprints/).

## Register the callback

Add `childBlueprintInputs` in the aggregate's init hook. Its functional interface:

```java
Map<String, Object> computeChildInputs(
    Map<String, Object> currentInputs,
    String blueprintName,
    Map<String, String> attributes);
```

It receives the enclosing blueprint's current inputs (including computed values) and the attribute map for that
specific marker, and must return a non-null map of the child's declared inputs. Required inputs must be present;
omitted optional inputs use the child's defaults.

```java
public void init(BlueprintConfig blueprint) {
    blueprint.childBlueprintInputs = (current, childName, attributes) -> {
        Map<String, Object> child = new LinkedHashMap<>();
        String version = current.get("applicationVersion").toString();
        String parentGroup = current.get("com") + "." + current.get("company")
                + "." + current.get("org") + "." + current.get("application");
        child.put("super_pom", Map.of(
                "groupId", parentGroup,
                "artifactId", current.get("application") + "-parent",
                "version", version,
                "relativePath", "../pom.xml"));
        switch (childName) {
            case "chenile-service" -> {
                child.put("service", attributes.get("name"));
                child.put("serviceVersion", version);
            }
            case "minimonolith" -> {
                String role = attributes.get("role");
                if (!Set.of("server", "worker").contains(role))
                    throw new IllegalArgumentException("Unknown mini-monolith role: " + role);
                child.put("monolith", role);
                child.put("monolithVersion", version);
            }
            default -> throw new IllegalArgumentException("Unexpected child: " + childName);
        }
        return child;
    };
}
```

No aggregate flag or extra JSON hook is needed — the existing `initHook` registers the callback. Included
blueprint modules must be on the CLI or portal classpath so their descriptors enter the registry.

## Include a child conditionally

Enclose the marker in a standard conditional folder and declare the flag as a `BOOLEAN` input:

```text
__application__/
  %%includeWorker=true%%/
    %{{minimonolith role=worker}}/   .gitkeep
```

When the flag is false the whole section is removed **before** expansion — JGen never resolves that child,
evaluates its attributes, invokes the callback, or generates its files. This holds even when the marker is
several directories below the conditional folder.

## Repeat a child for a prepared list

```text
%{{chenile-process-workers forEach=processes as=process}}/   .gitkeep
```

`forEach` and `as` are reserved expansion attributes: `forEach` names a list in the enclosing input map (dotted
paths are supported); `as` names the per-element binding. Each invocation gets an isolated map with
`currentInputs.get("process")` set to its element, and the callback computes the child's usual inputs. A marker
without these attributes invokes once. An empty list removes the marker without invoking the callback; a missing
or mistyped list, an invalid alias, and `as` without `forEach` are errors. All iterations are generated and
checked for collisions before the marker's outputs are published, and conditional folders can enclose loop
markers too.

## Lifecycle and isolation

1. The enclosing blueprint captures and validates inputs and runs its input hook.
2. Its templates render, filenames resolve, and conditional folders are selected.
3. JGen finds the surviving markers and calls `childBlueprintInputs` for each.
4. Each child runs its own complete pipeline in a staging directory.
5. JGen checks for output collisions, moves the generated tree beside the marker, and removes the marker.
6. The enclosing blueprint's post-process hook runs after all children.

A child may itself be an aggregate and uses its own callback. Inclusion cycles such as `A → B → A` are rejected.
Maps and lists passed to and returned from the callback are deep-copied, so children never share mutable state
with the parent or each other; attributes are read-only strings, and child output is never re-rendered with the
ancestor's inputs. Unknown child names, missing callbacks, invalid child inputs, non-empty marker folders,
recursion and output collisions all fail generation with identifying errors, and existing files are never merged
or overwritten.

> Generation is **not** transactional across the whole aggregate: parent files and earlier successful children
> can remain after a later child fails. Use a fresh output destination when retrying.
