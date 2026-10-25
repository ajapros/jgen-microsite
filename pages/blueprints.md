---
title: "Blueprint catalogue"
eyebrow: "Blueprints"
permalink: /blueprints/
summary: "The fourteen built-in blueprints — what each generates, its key inputs, and the minimum Chenile version it targets."
---

Each blueprint name below is the value you pass to `-g` and put in the `blueprint` field of an input file. The
safest way to learn any one of them is `jgen -g <name> -o input.json`, which writes a sample showing every field
it expects. Every blueprint also accepts an optional `super_pom` and `dependencies` (see
[Composable blueprints](/composable-blueprints/)); those two are omitted from the field lists below.

| Blueprint | Generates | Since |
| --- | --- | --- |
| [`chenile-service`](#chenile-service) | An HTTP Chenile service (`-api` + `-service`) | any |
| [`chenile-headless-service`](#chenile-headless-service) | A non-HTTP Chenile service | 2.1.31 |
| [`chenile-ecosystem`](#chenile-ecosystem) | A composed multi-project application | 2.1.31 |
| [`wfservice`](#wfservice) | A workflow (STM) service | any |
| [`wfcustom`](#wfcustom) | A workflow service from your own states XML | any |
| [`mybatisQuery`](#mybatisquery) | A MyBatis query module | any |
| [`minimonolith`](#minimonolith) | A runnable mini-monolith host | any |
| [`chenile-interceptor`](#chenile-interceptor) | A Chenile interceptor policy | any |
| [`batch`](#batch) | A batch process from a batch JSON | any |
| [`chenile-process-workers`](#chenile-process-workers) | One process definition's worker JARs | 2.1.32 |
| [`chenile-process-worker-set`](#chenile-process-worker-set) | Per-definition worker modules for a whole spec | 2.1.32 |
| [`chenile-process-management`](#chenile-process-management) | A process server + worker deployment | 2.1.32 |
| [`it`](#it) | An integration-test project | any |
| [`jgen-blueprint`](#jgen-blueprint) | The skeleton of a new blueprint module | any |

---

## chenile-service

Generates `<svc>-api` and `<svc>-service` (controller, impl, configuration, health checker, error codes, BDD
tests, `scripts/curl-scripts.sh`). Endpoints `POST /<svc>`, `GET /<svc>/{id}`, `POST /<svc>/op1`.

**Fields:** `service` *(JAVA_IDENTIFIER)*, `serviceVersion` *(VERSION)*, `destFolder`, `security` *(bool)*,
`jpa` *(bool)*, `cloudSwitchEnabled` *(bool)*, `enableMCP` *(bool)*.

## chenile-headless-service <span class="pill v">2.1.31</span>

A Spring-managed Chenile controller **without** `@RestController` or a `chenile-http` dependency: it uses
`@ChenileOperation` and `@ChenileBody`, so Chenile's core runtime can invoke it or it can receive events with no
HTTP endpoint. `registerInServiceRegistry` sets the generated `@ChenileController(registerInServiceRegistry = …)`
value — default `y`; choose `n` for ephemeral, event-only, or serverless services that stay in the local Chenile
configuration but must not be published to the remote registry.

**Fields:** `service` *(JAVA_IDENTIFIER)*, `serviceVersion` *(VERSION)*, `destFolder`,
`registerInServiceRegistry` *(bool)*.

## chenile-ecosystem <span class="pill v">2.1.31</span>

Composes the established blueprints rather than keeping a second set of templates. It always generates an HTTP
`chenile-service` and the primary `minimonolith` that packages it, then optionally adds: a separate
service-registry mini-monolith (plus registry delegates), cconfig in the primary monolith, a headless service
and a monolith that packages it, and a query service and a monolith with the query controller. JGen creates one
`<ecosystem>` folder whose `pom.xml` is the Maven reactor parent for every generated project. It is an
[aggregated blueprint](/aggregated-blueprints/).

**Fields:** `ecosystem` *(PROJECT_NAME)*, `ecosystemVersion` *(VERSION)*, `destFolder`, `service`
*(JAVA_IDENTIFIER)*, `monolith` *(JAVA_IDENTIFIER)*, `security` *(bool)*, `jpa` *(bool)*,
`includeServiceRegistry` *(bool)* + `serviceRegistryMonolith` + `serviceRegistryUrl` *(HTTP_URL)*,
`includeCconfig` *(bool)*, `includeHeadlessService` *(bool)* + `headlessService` + `headlessMonolith`,
`includeQueryService` *(bool)* + `queryNamespace` + `queryMonolith`.

## wfservice

A workflow (state-machine) service `OPENED → ASSIGNED → RESOLVED → CLOSED` (with `activity`, mandatory
activities lead to `ARCHIVED`). Generates the states XML, actions, entity store, and a per-service
`StateEntityInfoController`. Endpoints `POST /<svc>`, `GET /<svc>/{id}`, `PATCH /<svc>/{id}/{eventID}`; plus
introspection endpoints under `/<svc>/info/…` for state diagram, allowed actions, workflow JSON, testcases,
testcase visualization, and testcase state diagrams (the diagram endpoints return PNG bytes so an admin UI can
render them directly).

**Fields:** `service` *(PROJECT_NAME)*, `serviceVersion` *(VERSION)*, `destFolder`, `activity` *(bool)*,
`enablement` *(bool)*, `jpa` *(bool)*, `cloudSwitchEnabled` *(bool)*, `security` *(bool)*, `enableMCP` *(bool)*.

## wfcustom

The same as `wfservice`, but driven from **your own** workflow states XML (`xmlFile`). Its init hook parses that
XML and injects the workflow and testcase structures into the template map. This is the "Finito" blueprint:
turn a hand-written flow into `-api` and `-service` modules.

**Fields:** `service` *(PROJECT_NAME)*, `serviceVersion` *(VERSION)*, `destFolder`, `jpa` *(bool)*,
`cloudSwitchEnabled` *(bool)*, `xmlFile` *(FILE)*, `security` *(bool)*, `activity` *(bool)*, `enablement`
*(bool)*, `genCustomizedPayload` *(bool)*, `genDefaultPostSaveHook` *(bool)*, `genDefaultAutoState` *(bool)*,
`enableMCP` *(bool)*.

## mybatisQuery

Generates `mapper/<ns>.xml` (with the `getAll-count` query and `${pagination}`) and `mapper/<ns>.json`, served
at `POST /q/<name>` by `chenile-query-controller`.

**Fields:** `namespace` *(JAVA_IDENTIFIER)*, `namespaceVersion` *(VERSION)*, `destFolder`, `security` *(bool)*.

## minimonolith

A runnable Spring-Boot mini-monolith host. `application.yml` ships with Keycloak at `localhost:8180` (client
`authz-servlet`) and MCP at `/mcp`. It is reused as a child by `chenile-ecosystem` and
`chenile-process-management` to produce their runnable hosts, and accepts host-shaping inputs such as
`runtimeProperties`, `scanPackages`, `entityPackages` and `repositoryPackages`.

**Fields:** `monolith` *(JAVA_IDENTIFIER)*, `monolithVersion` *(VERSION)*, `destFolder`, `jpa` *(bool)*,
`cloudSwitchEnabled` *(bool)*, `security` *(bool)*, `enableMCP` *(bool)* + `mcpServerName` + `mcpInstructions`,
`enableServiceRegistry` *(bool)*, `enableServiceRegistryDelegate` *(bool)*, `enableCconfig` *(bool)*,
`serviceRegistryUrl` *(HTTP_URL)*, `enableQueryController` *(bool)*, `enableH2Console` *(bool)*, `deploymentUrl`
*(HTTP_URL)*, `runtimeProperties`/`scanPackages`/`entityPackages`/`repositoryPackages` *(record arrays)*.

## chenile-interceptor

Scaffolds a `BaseChenileInterceptor` policy.

**Fields:** `interceptorName` *(JAVA_IDENTIFIER)*, `interceptorVersion` *(VERSION)*, `destFolder`.

## batch

Generates a batch process from a batch-definition JSON (`batchJson`); its init hook parses the file and computes
root and child process metadata before generation. It is built against the consolidated `process-delegate` and
tests over Kafka with `KafkaBaseTest` (Testcontainers). Prefer `chenile-process-management` for new process
work.

**Fields:** `batch` *(JAVA_IDENTIFIER)*, `batchVersion` *(VERSION)*, `destFolder`, `batchJson` *(FILE)*.

## chenile-process-workers <span class="pill v">2.1.32</span>

Produces the **API + service JARs for one process definition**: a **leaf** type gets an *executor*; a
**non-leaf** type gets a *splitter + aggregator* in the same JAR. A plain JAR (no Spring host); its payload model
comes from the application's API JAR. Since 2.1.35 the generated workers extend the typed
`ExecutorWorkListener` / `SplitterWorkListener` / `AggregatorWorkListener` bases (executors and splitters implement
`doStart(WorkListenerInput<In>)`; the aggregator folds one child at a time), and their `doStart` hooks return
defaults until you implement them — they are application code. Worker bean names double as routing keys
(`<processType>Executor|Splitter|Aggregator`): the generated worker listener uses chenile-core's `serviceSelector`
to pick the bean for each message. You rarely invoke this directly: `chenile-process-worker-set` calls it once
per definition.

**Fields:** `workerModule` *(PROJECT_NAME)*, `application` *(JAVA_IDENTIFIER)*, `applicationVersion` *(VERSION)*,
`packageName` *(JAVA_PACKAGE)*, `processType` *(JAVA_IDENTIFIER)*, `name` *(JAVA_IDENTIFIER)*, `inputType`
*(JAVA_IDENTIFIER)*, `fields` *(records)*, `outputType`, `childOutputType`, `outputFields` *(records)*,
`apiArtifact` *(PROJECT_NAME)*, `serviceArtifact` *(PROJECT_NAME)*, `leaf` *(bool)*, `children` *(records)*,
`destFolder`.

## chenile-process-worker-set <span class="pill v">2.1.32</span>

Takes a complete, prepared process-definition set and **expands it into per-definition worker modules**,
invoking `chenile-process-workers` once per process type. It owns the iteration (and the process-definition
validation) so the process-management aggregate stays simple. You rarely run it standalone —
`chenile-process-management` invokes it once with the marker `%{{chenile-process-worker-set}}`. It is an
[aggregated blueprint](/aggregated-blueprints/).

**Fields:** `application`, `applicationVersion`, `com`/`company`/`org`, `destFolder`, `processSpec` *(FILE)*,
`definitionSource` *(dropdown)*.

## chenile-process-management <span class="pill v">2.1.32</span>

The **deployment generator**. From native `processMap` definitions it emits a process **server** mini-monolith,
a headless **worker** mini-monolith, **per-definition worker JARs**, and an optional **service-registry** host —
by composing `minimonolith` with `chenile-process-worker-set`, decoupled over Kafka. It is an
[aggregated blueprint](/aggregated-blueprints/). Full guide: `bp-process-management/README.md` and the
[process deployment tutorial](https://chenile.org/developer/process-deployment-tutorial/) on chenile.org.

**Fields:** `application` *(JAVA_IDENTIFIER)*, `applicationVersion` *(VERSION)*, `destFolder`, `processSpec`
*(FILE)*, `cronSpec` *(FILE, optional)*, `definitionSource` *(dropdown: `json`|`database`)*, `registryMode`
*(dropdown: `create`|`existing`)* + `existingRegistryUrl` *(HTTP_URL)*, `registryPlacement` *(dropdown:
`server`|`separate`)*, `serverDeploymentUrl`/`workerDeploymentUrl`/`registryDeploymentUrl` *(HTTP_URL)*,
`transport` *(dropdown: `kafka`)*, `workTopic` *(KAFKA_TOPIC)*.

## it

An integration-test project using `it-cucumber-utils`. The entity input uses `PROJECT_NAME` so hyphenated entity
names survive into route and feature filenames.

**Fields:** `app` *(JAVA_IDENTIFIER)*, `appVersion` *(VERSION)*, `entity` *(PROJECT_NAME)*, `destFolder`,
`security` *(bool)*.

## jgen-blueprint

Generates the **skeleton of a new blueprint module** — a Maven module, a `blueprint.json` descriptor, an init
hook class, and a Java package. It does not generate a finished blueprint; it gives you the shell to finish by
hand. See [Write your own blueprint](/authoring/) for the full workflow.

**Fields:** `blueprintName` *(JAVA_IDENTIFIER)*, `destFolder`, `security` *(bool)*, `jpa` *(bool)*,
`cloudSwitchEnabled` *(bool)*.
