---
title: "jgen(1): the manual"
eyebrow: "Reference"
permalink: /manual/
summary: "Every jgen option, mode, environment variable and file, in the style of a Unix manual page. Run man jgen after a Homebrew or zip install."
---

<p>This is the manual page installed with JGen (<code>man jgen</code>). Its source is
<code>jgen/jgen-cli/src/man/jgen.1.md</code> in the chenile-gen repository.</p>

{% raw %}
<div class="manpage">
<h2 id="name">NAME</h2>
<p>jgen - generate projects from blueprints</p>
<h2 id="synopsis">SYNOPSIS</h2>
<p><strong>jgen</strong> [<strong>-c</strong> <em>config</em>]</p>
<p><strong>jgen</strong> <strong>-f</strong> <em>input.json</em> [<strong>-c</strong> <em>config</em>] [<strong>--trust</strong>] [<strong>--offline</strong>]</p>
<p><strong>jgen</strong> <strong>-g</strong> <em>blueprint</em> [<strong>-o</strong> <em>file</em> | <strong>-e</strong> <em>folder</em>] [<strong>-c</strong> <em>config</em>] [<strong>--trust</strong>] [<strong>--offline</strong>]</p>
<p><strong>jgen</strong> <strong>-e</strong> <em>config-file</em></p>
<p><strong>jgen</strong> <strong>-r</strong> <em>project</em> [<strong>--into</strong> <em>folder</em>] [<strong>--blueprint</strong> <em>blueprint</em>] [<strong>-c</strong> <em>config</em>] [<strong>--trust</strong>] [<strong>--offline</strong>]</p>
<p><strong>jgen</strong> <strong>--fetch</strong> <em>reference</em> [<strong>--trust</strong>] [<strong>--offline</strong>]</p>
<p><strong>jgen</strong> <strong>--completion</strong> <em>shell</em></p>
<p><strong>jgen</strong> <strong>-h</strong> | <strong>-V</strong></p>
<h2 id="description">DESCRIPTION</h2>
<p><strong>jgen</strong> generates a project from a <em>blueprint</em>. A blueprint is one jar that holds a typed input contract (<em>META-INF/blueprint.json</em>), a template tree, and optional Java hooks. jgen collects the blueprint’s inputs, checks them, copies the template tree into the destination folder, renders it, and runs the blueprint’s hooks.</p>
<p>Template files ending in <em>.mustache</em> are rendered with Mustache. File and folder names carry further instructions:</p>
<ul>
<li><em>__name__</em> is replaced by an input value;</li>
<li><em>_DoT_</em> becomes a dot, so <em>_DoT_gitignore</em> is generated as <em>.gitignore</em>;</li>
<li>a folder named <em>%%key=value%%</em> is kept only when the input matches;</li>
<li>a <em>.filelist</em> file is split into several files;</li>
<li>an empty folder named <em>%{{blueprint …}}</em> generates a child blueprint in its place.</li>
</ul>
<p>jgen ships with blueprints for Chenile services, workflow services, query modules, mini-monoliths, ecosystems, process deployments, interceptors, integration tests and new blueprints. It accepts other blueprints without a rebuild: from blueprint folders (see <strong>FILES</strong>) or straight from a GitHub release (see <strong>BLUEPRINT REFERENCES</strong>).</p>
<p>Every input path (prompts, input files, the web portal, child blueprints) goes through the same validator. The validator reports every problem at once, before any file is written.</p>
<p>A blueprint can ask jgen to make the generated project a local Git repository: when the input <strong>initGit</strong> is true, jgen runs <strong>git init</strong> (branch <strong>main</strong>), adds every file and makes one commit. If the blueprint names a <strong>GITHUB_REPOSITORY</strong> input as its <em>gitRepositoryField</em> and that input has a value, jgen adds it as the <strong>origin</strong> remote, using SSH, or HTTPS when the configuration sets <strong>gitRemoteProtocol</strong> to <strong>https</strong>. jgen never creates the GitHub repository and never pushes.</p>
<p>After generating, jgen writes a generation record into the project (see <strong>FILES</strong>). <strong>jgen -r</strong> uses it to regenerate the project with a newer blueprint and report what changed.</p>
<h2 id="modes">MODES</h2>
<p>jgen does one thing per run. When several mode options are given, the first of <strong>--fetch</strong>, <strong>-r</strong>, <strong>-g</strong> and <strong>-e</strong> wins. With none of them, jgen generates a project.</p>
<dl>
<dt><strong>Interactive generation</strong> (no mode option)</dt>
<dd>If a <em>config</em> folder exists in the current directory, jgen offers its files as alternatives to the default configuration. It then lists the installed blueprints, asks you to choose one, and prompts for each input. Each prompt shows the input’s validation rule and default value. Press Enter to accept the default.
</dd>
<dt><strong>Generation from a file</strong> (<strong>-f</strong>)</dt>
<dd>Reads all inputs from a JSON file and asks nothing. The whole file is validated first, and every error is listed. Keys that the blueprint does not declare are reported as warnings and ignored.
</dd>
<dt><strong>Sample input</strong> (<strong>-g</strong>)</dt>
<dd>Writes a JSON input file for a blueprint, filled with its default values, ready to edit and pass to <strong>-f</strong>.
</dd>
<dt><strong>Configuration</strong> (<strong>-e</strong> without <strong>-g</strong>)</dt>
<dd>Writes a copy of the default configuration to edit and pass to <strong>-c</strong>.
</dd>
<dt><strong>Regeneration</strong> (<strong>-r</strong>)</dt>
<dd>Regenerates a project with the current blueprint, into a separate folder, and classifies every file. The original project is never modified.
</dd>
<dt><strong>Fetch</strong> (<strong>--fetch</strong>)</dt>
<dd>Fetches, verifies and caches a blueprint from GitHub, and describes it.
</dd>
</dl>
<h2 id="options">OPTIONS</h2>
<dl>
<dt><strong>-c</strong>, <strong>--config</strong> <em>file</em></dt>
<dd>Use this JGen configuration instead of the bundled default. The configuration sets the Java package segments (<em>com</em>, <em>company</em>, <em>org</em>), the Chenile version used in generated projects, and defaults such as the destination folder. A missing file is an error. With <strong>-r</strong>, use this configuration instead of the one recorded in the project.
</dd>
<dt><strong>-f</strong>, <strong>--input-file</strong> <em>file</em></dt>
<dd>Generate from the inputs in <em>file</em>, a JSON object. Its <strong>blueprint</strong> key names the blueprint (an installed name or a <strong>github:</strong> reference); its other keys are the blueprint’s inputs. Values may refer to configuration entries with <em>${name}</em>.
</dd>
<dt><strong>-g</strong>, <strong>--generate-blueprint-file</strong> <em>blueprint</em></dt>
<dd>Emit a sample input file for <em>blueprint</em>, an installed name or a <strong>github:</strong> reference. The sample keeps the reference in its <strong>blueprint</strong> key, so <strong>-f</strong> later resolves the same blueprint. Without <strong>-o</strong> or <strong>-e</strong>, the sample is printed on standard output.
</dd>
<dt><strong>-o</strong>, <strong>--output-file</strong> <em>file</em></dt>
<dd>With <strong>-g</strong>: write the sample to <em>file</em>. It cannot be combined with <strong>-e</strong> <em>folder</em>.
</dd>
<dt><strong>-e</strong>, <strong>--emit-config</strong> [<em>name</em>]</dt>
<dd><p>Without <strong>-g</strong>: write the default configuration to <em>config/name.json</em>. The <em>config/</em> folder is created if needed, and <em>.json</em> is added if missing.</p>
<p>With <strong>-g</strong>: <em>name</em> is a folder. jgen writes <em>input-file.json</em> there, together with any companion sample files the blueprint provides (for example a workflow XML or a process definition). It refuses to overwrite existing companion files.</p>
</dd>
<dt><strong>-r</strong>, <strong>--regenerate</strong> <em>project</em></dt>
<dd><p>Regenerate <em>project</em>, a folder that contains <em>.jgen/generation.json</em>. jgen finds the blueprint by its recorded name. If that blueprint is not installed, it fetches exactly the recorded reference and checks it against the recorded hash. jgen then validates the recorded inputs against the blueprint’s current input contract and generates into <strong>--into</strong>. Each file is reported in one of these classes:</p>
<p><strong>SAFE_UPDATE</strong> - you did not change it, the blueprint did: take the new file.<br />
<strong>KEEP_USER</strong> - you changed it, the blueprint did not: keep yours.<br />
<strong>CONFLICT</strong> - changed on both sides: merge.<br />
<strong>NEW_FILE</strong> - new in the blueprint: add it.<br />
<strong>SAFE_DELETE</strong> - dropped by the blueprint, unchanged by you: delete it.<br />
<strong>DELETE_CONFLICT</strong> - dropped by the blueprint, changed by you: decide.<br />
<strong>USER_DELETED</strong> - you deleted it, the blueprint still generates it.</p>
<p>jgen prints the classification, writes it to <em>project.regeneration.json</em> next to the regenerated project, and suggests a <strong>git diff --no-index</strong> command.</p>
</dd>
<dt><strong>--into</strong> <em>folder</em></dt>
<dd>With <strong>-r</strong>: the folder to regenerate into. Default: <em>project.regenerated</em>, next to <em>project</em>.
</dd>
<dt><strong>--blueprint</strong> <em>blueprint</em></dt>
<dd>With <strong>-r</strong>: regenerate with this blueprint instead of the recorded one, for example <strong>github:acme/jgen-terraform@2.0.0</strong>, to see what an upgrade would change. It must have the same name as the recorded blueprint.
</dd>
<dt><strong>--fetch</strong> <em>reference</em></dt>
<dd>Fetch the blueprint at <em>reference</em>, verify it and cache it. jgen prints its name, description, exact source and SHA-256, and the <strong>-g</strong> command to use it.
</dd>
<dt><strong>--trust</strong></dt>
<dd>Trust every blueprint source used in this run without asking. Use this in scripts and CI only for sources you have reviewed. Prefer listing them in <em>settings.json</em> (see <strong>FILES</strong>).
</dd>
<dt><strong>--offline</strong></dt>
<dd>Never contact GitHub. Use only installed and cached blueprints. A reference without a version resolves to the newest cached version.
</dd>
<dt><strong>--completion</strong> <em>shell</em></dt>
<dd>Print the completion script for <em>shell</em>: <strong>bash</strong>, <strong>zsh</strong> or <strong>fish</strong> (see <strong>SHELL COMPLETION</strong>).
</dd>
<dt><strong>-h</strong>, <strong>--help</strong></dt>
<dd>Print a usage summary and exit.
</dd>
<dt><strong>-V</strong>, <strong>--version</strong></dt>
<dd>Print the version and exit.
</dd>
</dl>
<h2 id="blueprint-references">BLUEPRINT REFERENCES</h2>
<p>A blueprint is named either by its installed name (for example <strong>chenile-service</strong>) or by a reference:</p>
<p><strong>github:</strong><em>owner</em>/<em>repo</em>[/<em>name</em>][@<em>version</em>][#sha256:<em>hex</em>]</p>
<dl>
<dt><em>owner/repo</em></dt>
<dd>The GitHub repository. Its release must carry exactly one blueprint jar, or exactly one named <em>repo-*.jar</em>. Jars ending in <em>-sources</em>, <em>-javadoc</em> and <em>-tests</em> are ignored.
</dd>
<dt><em>name</em></dt>
<dd>Which blueprint to use, when the jar contains several.
</dd>
<dt><em>version</em></dt>
<dd>A release tag. Without it, jgen uses the latest release (or, with <strong>--offline</strong>, the newest cached one).
</dd>
<dt><em>hex</em></dt>
<dd>The expected SHA-256 of the jar, as 64 hexadecimal digits. A mismatch is an error. Pinning is recommended in scripts and CI.
</dd>
</dl>
<p>Installed blueprints always take precedence. A fetched blueprint whose name is already taken by an installed one is refused.</p>
<h2 id="trust">TRUST</h2>
<p>A blueprint can contain Java code, which runs with your permissions. jgen therefore:</p>
<ul>
<li>downloads only from trusted sources. By default these are the official organisations, <strong>github:ajapros/*</strong> and <strong>github:rajakolluru/*</strong>. For any other source jgen asks first: allow once, allow always (which adds the repository to <em>settings.json</em>), or refuse. Without a terminal it refuses unless the source is listed or <strong>--trust</strong> is given. Nothing is downloaded before trust is granted.</li>
<li>records the SHA-256 of each fetched version in <em>blueprints.lock.json</em> the first time it is used, and refuses the version if its jar later changes.</li>
<li>loads each fetched jar in its own class loader.</li>
</ul>
<h2 id="shell-completion">SHELL COMPLETION</h2>
<p>Homebrew installs completion for bash, zsh and fish. Otherwise, enable it once:</p>
<dl>
<dt>bash (add to <em>~/.bashrc</em>)</dt>
<dd><strong>source &lt;(jgen --completion bash)</strong>
</dd>
<dt>zsh (add to <em>~/.zshrc</em>, after <strong>compinit</strong>)</dt>
<dd><strong>source &lt;(jgen --completion zsh)</strong>
</dd>
<dt>fish</dt>
<dd><strong>jgen --completion fish &gt; ~/.config/fish/completions/jgen.fish</strong>
</dd>
</dl>
<p>Zip installs also carry the scripts in <em>completions/</em>.</p>
<p>Tab completes options. For <strong>-g</strong>, <strong>--blueprint</strong> and <strong>--fetch</strong> it offers the installed blueprints, cached <strong>github:</strong> references and the <strong>github:</strong> prefix. For <strong>-f</strong> and <strong>-c</strong> it offers JSON files, and for <strong>-r</strong> and <strong>--into</strong> it offers folders. The blueprint list is cached for ten minutes, or until a jar is added to <em>~/.jgen/blueprints</em>, so Tab does not start Java each time.</p>
<h2 id="exit-status">EXIT STATUS</h2>
<dl>
<dt><strong>0</strong></dt>
<dd>Success.
</dd>
<dt><strong>1</strong></dt>
<dd>An error occurred. This includes invalid inputs (every problem is listed), an unknown or untrusted blueprint, a hash mismatch, a missing configuration file, a destination folder that already exists, and a failure in a blueprint hook. Input and trust errors stop jgen before it writes anything.
</dd>
</dl>
<h2 id="environment">ENVIRONMENT</h2>
<dl>
<dt><strong>JAVA_HOME</strong></dt>
<dd>The Java runtime to use (Java 25 or later). Otherwise <strong>java</strong> is taken from the PATH, or from Homebrew’s OpenJDK.
</dd>
<dt><strong>JAVA_OPTS</strong></dt>
<dd>Extra JVM options. <strong>-Djgen.record=false</strong> turns off generation records.
</dd>
<dt><strong>JGEN_BLUEPRINTS</strong></dt>
<dd>Extra blueprint folders, comma-separated. Every jar in them is loaded at start-up.
</dd>
<dt><strong>JGEN_HOME</strong></dt>
<dd>The installation folder. The launcher sets it.
</dd>
<dt><strong>JGEN_HOME_DIR</strong></dt>
<dd>Use this folder instead of <em>~/.jgen</em> for settings, the lock file and the cache. Useful in CI.
</dd>
<dt><strong>GITHUB_TOKEN</strong>, <strong>GH_TOKEN</strong></dt>
<dd>A GitHub token, used to raise the API rate limit when fetching blueprints.
</dd>
</dl>
<h2 id="files">FILES</h2>
<dl>
<dt><em>~/.jgen/blueprints/</em></dt>
<dd>Your blueprint jars. They are available to every jgen run, including runs through JBang.
</dd>
<dt><em>$JGEN_HOME/blueprints/</em></dt>
<dd>Blueprint jars shared by everyone using this installation.
</dd>
<dt><em>~/.jgen/settings.json</em></dt>
<dd><strong>trustedSources</strong>: a list of trusted blueprint sources, each <strong>github:</strong><em>owner</em>/* or <strong>github:</strong><em>owner</em>/<em>repo</em>.
</dd>
<dt><em>~/.jgen/blueprints.lock.json</em></dt>
<dd>The SHA-256 first seen for each fetched blueprint version.
</dd>
<dt><em>~/.jgen/cache/github/owner/repo/version/</em></dt>
<dd>Cached blueprint jars.
</dd>
<dt><em>config/</em></dt>
<dd>Configuration files, written by <strong>-e</strong> and offered by interactive generation.
</dd>
<dt><em>~/.cache/jgen/blueprint-names</em></dt>
<dd>The blueprint list used by shell completion (under <em>$XDG_CACHE_HOME</em> if set). It is safe to delete.
</dd>
<dt><em>project/.jgen/generation.json</em></dt>
<dd><p>The generation record. It holds:</p>
<ul>
<li>the blueprint (name, version, jar coordinates, and for fetched blueprints its reference and SHA-256);</li>
<li>the configuration;</li>
<li>the validated inputs, with copies of file inputs under <em>.jgen/inputs/</em>;</li>
<li>the SHA-256 of every generated file.</li>
</ul>
<p>Commit it with the project.</p>
</dd>
<dt><em>project.regeneration.json</em></dt>
<dd>The classification written by <strong>-r</strong>.
</dd>
</dl>
<h2 id="examples">EXAMPLES</h2>
<p>Generate a Chenile service from an edited sample:</p>
<pre><code>jgen -g chenile-service -o orders.json
jgen -f orders.json</code></pre>
<p>Emit a workflow blueprint’s input together with its sample workflow XML:</p>
<pre><code>jgen -g wfcustom -e specs/
jgen -f specs/input-file.json</code></pre>
<p>Use a blueprint published on GitHub, then pin it for CI (<strong>--fetch</strong> prints the hash):</p>
<pre><code>jgen --fetch github:acme/jgen-terraform@1.2.0
jgen -g github:acme/jgen-terraform@1.2.0#sha256:&lt;hex&gt; -o tf.json
jgen -f tf.json --trust</code></pre>
<p>Start a new blueprint in its own Git repository, ready to push (set <strong>standaloneRepo</strong> to <strong>y</strong> in the input):</p>
<pre><code>jgen -g jgen-blueprint -o new-blueprint.json
jgen -f new-blueprint.json</code></pre>
<p>Install a blueprint jar of your own:</p>
<pre><code>cp target/bp-docs-site-1.0.jar ~/.jgen/blueprints/
jgen -g docs-site -o docs.json</code></pre>
<p>See what a newer blueprint would change in an existing project:</p>
<pre><code>jgen -r ./output/orders
jgen -r ./output/infra --blueprint github:acme/jgen-terraform@2.0.0
git diff --no-index ./output/orders ./output/orders.regenerated/orders</code></pre>
<p>Work without network access:</p>
<pre><code>jgen -g github:acme/jgen-terraform --offline -o tf.json</code></pre>
<h2 id="see-also">SEE ALSO</h2>
<p><strong>java</strong>(1), <strong>git-diff</strong>(1), <strong>mvn</strong>(1)</p>
<p>The JGen documentation: <a href="https://jgen.chenile.org" class="uri">https://jgen.chenile.org</a>. Chenile: <a href="https://chenile.org" class="uri">https://chenile.org</a>.</p>
<h2 id="bugs">BUGS</h2>
<p>Report problems at <a href="https://github.com/rajakolluru/chenile-gen/issues" class="uri">https://github.com/rajakolluru/chenile-gen/issues</a>.</p>
<h2 id="author">AUTHOR</h2>
<p>Raja Shankar Kolluru and the Chenile contributors.</p>
<h2 id="copyright">COPYRIGHT</h2>
<p>MIT License. Copyright (c) 2019 Raja Shankar Kolluru.</p>
</div>
{% endraw %}
