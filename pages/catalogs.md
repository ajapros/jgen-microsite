---
title: "Blueprint catalogs"
eyebrow: "Using JGen"
permalink: /catalogs/
summary: "Use blueprints by short name: jgen -g terraform. Catalogs map names to blueprint sources; the official catalog is built in, and teams can add their own."
---

<span class="pill">2.5.1</span>

A `github:` reference names a blueprint precisely, but you have to know it. A **catalog** maps short names to
those references, so people can write what they mean:

```bash
jgen --search terraform                 # find blueprints by name, description or tag
jgen -g terraform -o tf.json            # the latest version
jgen -g terraform@1.2.0 -o tf.json      # a specific version
jgen -g acme/terraform -o tf.json       # the entry in the catalog called "acme"
```

## How a name is resolved

1. **Installed blueprints first.** A plain name that matches a built-in blueprint, or a jar in a blueprints folder,
   uses that blueprint. Installed blueprints always win.
2. **Then the catalogs, in order:** the official catalog, then your own, in the order listed in settings. The first
   catalog with the name wins. `alias/name` picks a specific catalog.
3. **The entry's source is fetched** exactly like a `github:` reference: a release jar, or a template-only
   repository at a tag ([blueprints from the web](/remote-blueprints/)). `name@version` picks the version;
   without it, the latest.

The generation record stores the resolved `github:` reference and hash, not the short name, so
[`jgen -r`](/regeneration/) still fetches exactly the blueprint that cut the project.

## The official catalog

It lives in [ajapros/jgen-catalog](https://github.com/ajapros/jgen-catalog) as `catalog.json`. Listing there is an
endorsement: every entry is reviewed, so its sources are **trusted** without a prompt. To add a blueprint, open a
pull request against that repository; its check validates the entry.

## Your own catalog

Any GitHub repository with a `catalog.json` at the root of its default branch is a catalog. List it in
`~/.jgen/settings.json`:

```json
{
  "catalogs": [
    { "source": "github:acme/jgen-catalog", "alias": "acme", "trusted": false }
  ],
  "officialCatalog": true
}
```

- **`alias`** is the name used in `acme/terraform`. It defaults to the repository owner.
- **`trusted`** controls whether the sources the catalog lists are trusted without asking. Leave it `false` unless
  you control the catalog. With `false`, each source goes through the normal [trust](/remote-blueprints/#trust)
  prompt.
- **`officialCatalog`** set to `false` searches only your catalogs.

## The catalog format

```json
{
  "schema": "jgen.catalog/1",
  "description": "Blueprints curated by Acme",
  "blueprints": {
    "terraform": {
      "source": "github:acme/jgen-terraform",
      "description": "A Terraform module with a CI workflow",
      "tags": ["terraform", "iac"],
      "versions": { "1.2.0": "3f9c…64 hex digits…" },
      "blueprint": "terraform-module"
    }
  }
}
```

| Key | Meaning |
| --- | --- |
| `source` | required: `github:owner/repo`, a jar release or a template-only repository |
| `description`, `tags` | shown and matched by `jgen --search` |
| `versions` | optional pins: a listed version's SHA-256 is enforced like a `#sha256:` pin (`jgen --fetch` prints it) |
| `blueprint` | optional: which blueprint to use when the source holds several |

Entries without a valid `github:` source are ignored with a warning.

## Caching and offline use

Catalogs are cached under `~/.jgen/cache/catalogs/` and refreshed after an hour. If GitHub can't be reached,
JGen uses the cached copy and says so. With `--offline` it uses only the cache. Tab completion also offers names
from cached catalogs, without going to the network.
