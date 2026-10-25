# jgen.chenile.org

The documentation microsite for **JGen**, Chenile's pure-Java, blueprint-driven code generator
(source: [rajakolluru/chenile-gen](https://github.com/rajakolluru/chenile-gen)).

A Jekyll site in the Chenile house style (Josefin Sans / Sora / JetBrains Mono, amber-on-black), deployed on
GitHub Pages. `CNAME` is `jgen.chenile.org`.

## Structure

- `index.html` — landing page (hero + feature strip + quickstart).
- `pages/*.md` — the docs, each with an explicit `permalink`. Sidebar order lives in `_data/docsnav.yml`.
- `_layouts/` — `default.html` (shell: top nav + footer) and `doc.html` (sidebar + article).
- `_includes/` — `head.html`, `nav.html`, `footer.html`.
- `assets/css/main.css` — the single stylesheet. `assets/img/` — brand marks copied from chenile.org.

## Pages

Overview · Install & the CLI · Configuration & versions · Blueprint catalogue · Template conventions ·
Generation pipeline · Write your own blueprint · Declarative input validation · Composable blueprints ·
Aggregated blueprints.

## Build locally

```bash
bundle install
bundle exec jekyll serve
```

## Keeping it accurate

The content is derived from the `chenile-gen` repo:

- `jgen/README.md`, `jgen/docs/{composable,aggregated}-blueprints.md`, `jgen/docs/input-validation.md`
- each `bp-*/.../META-INF/blueprint.json` (blueprint fields, `sinceVersion`, `templateFolder`)
- `jgen-base/.../template/processors.xml` (the processor chain order)
- the **Template reference** page is derived from the chain classes in
  `jgen-base/.../template/chain/`: `FileTemplateExecutor` + `LambdaLoader` (Mustache + the `capitalize` lambda),
  `FileNameReplacer` (`__key__` via commons-text `StringSubstitutor`), `ConditionalProcessor` (`%%k=v%%`),
  `FileSplitter` (`.filelist`), and `BlueprintMarker` / `BlueprintExpander` (`%{{…}}` grammar).

When JGen changes, re-check those sources. See `DEPLOY.md` for publishing.
