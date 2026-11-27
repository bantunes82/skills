---
name: quarkus-upgrade
description: Upgrade an existing Maven Quarkus application to a newer Quarkus release, one validated stage at a time, using the pinned OpenRewrite recipes and `quarkus update` commands in this skill. Use whenever someone wants to upgrade, migrate, bump or modernize Quarkus — e.g. 1.x → 2.x, 2.x → 3.x, an older 3.x → latest 3.x, 3.x → 4.x, moving off an end-of-life stream or fixing CVEs in Quarkus dependencies, running `quarkus update`, the javax → jakarta switch for a Quarkus app, or raising Java to 17 (Quarkus 3) or 21/25 (Quarkus 4) so a newer Quarkus can be used — even if the word "upgrade" never appears. Includes build and test failure repair, mandatory execution of EVERY run/validation approach documented in README.md, AGENTS.md, and CLAUDE.md after each stage, a mandatory smoke test of every interface the running application exposes (REST API, OpenAPI/Swagger UI, health, web UI, Dev UI, auth), and confirmation between stages. Not for creating new Quarkus projects, adding extensions, or upgrading other frameworks (Spring Boot, Micronaut, Jakarta EE servers).
compatibility: Requires Maven, JDK 11+ for Quarkus 2.x, JDK 17+ for Quarkus 3.x (the Quarkus 3.40 Maven plugin itself needs JDK 17 to run), and JDK 21+ (25 recommended) for Quarkus 4.x, access to the configured Maven repositories, and the application's documented validation dependencies.
---

# Quarkus upgrade

Carry out one validated upgrade stage at a time. A successful recipe or
`quarkus:update` run is only the start: compilation, unit/integration tests,
documented application checks, and a smoke test of the running application's
real interfaces after every step establish whether the application is ready.

## 1. Inspect the project

- Locate the repository root. Use the workflow and commands defined in this skill
  as the upgrade procedure; no separate workflow document is required.
- MANDATORY: locate and read every `README.md`, `AGENTS.md`, and `CLAUDE.md` file
  in the repository (root and every module; search the tree rather than assuming
  locations). Extract **every** documented step and approach for building,
  running, testing, and validating the application — whatever the heading
  ("Running the application in dev mode", "Packaging and running", "Creating a
  native executable", "Container image", "Testing", "Coverage", etc.). Treat these
  documents as the authoritative, non-optional validation procedure for this
  project — do not substitute a generic `mvn clean install` or a guessed
  `java -jar` invocation for them.
- Build the **documented validation checklist** now and keep it for the whole
  upgrade (see section 3, step 5). List each approach/step as its own item with:
  source file and heading, exact command(s) and working directory, prerequisites
  (Docker/Podman for Dev Services or Testcontainers, GraalVM/Mandrel or a
  container build for native, Compose, tokens…), and the success signal (exit
  code, readiness/health URL, endpoint response, report produced). When the same
  workflow is described in several files, merge into one item but keep every
  distinct command. Alternatives are *not* "pick one": if a README lists dev
  mode, JVM jar, über-jar, native executable, and container image, the checklist
  has all of them, plus every documented test, integration-test (`verify`,
  `-Dnative`), coverage, and analysis command.
- Build the **smoke-test plan** now as well (see section 3a). Documentation
  rarely lists every interface, so derive it from the code and configuration
  too (`application.properties`/`application.yaml`, including `%dev`, `%test`,
  `%prod` profile overrides):
  - base URL: `quarkus.http.port`, `quarkus.http.root-path`,
    `quarkus.http.non-application-root-path` (management endpoints live under
    `/q` since Quarkus 1.11/2.x; very old 1.x serves `/health`, `/openapi`
    without the prefix), and `quarkus.management.*` if a separate management
    interface is enabled;
  - endpoints: JAX-RS resources (`@Path`), reactive routes (`@Route`), RESTEasy
    Classic vs. RESTEasy Reactive / Quarkus REST, REST clients the app calls;
  - extensions in the POMs: SmallRye Health (`/q/health`, `/live`, `/ready`),
    SmallRye OpenAPI (`/q/openapi`) and Swagger UI (`/q/swagger-ui`, only in
    dev/test unless `quarkus.swagger-ui.always-include=true`), Micrometer or
    SmallRye Metrics (`/q/metrics`), GraphQL (`/graphql`), WebSockets, gRPC,
    Reactive Messaging (Kafka/AMQP), scheduler, Hibernate ORM/Panache with
    Dev Services or a real datasource;
  - web UI: static files in `src/main/resources/META-INF/resources/`, Qute
    templates in `src/main/resources/templates/`, Renarde/web bundler;
  - security model: `@RolesAllowed`/`@Authenticated`,
    `quarkus.http.auth.permission.*`, OIDC/JWT/basic auth, documented test
    users or token helpers.

  Record which interfaces exist; "no web UI" is a finding, not an assumption.
- Before running the first recipe, if the application can be started with the
  documented default approach, capture a **baseline smoke run** (in particular
  save the OpenAPI document and representative responses). Comparing against it
  after the upgrade exposes silently dropped endpoints, changed JSON shapes, or
  altered status codes that a bare "200 OK" would hide. If a baseline cannot be
  captured, say so and continue.
- Inspect the working tree so unrelated user changes are preserved.
- Inspect the root and module `pom.xml` files and resolve Quarkus's effective
  version. It may come from `quarkus.platform.version`/`quarkus.version`
  properties, an imported `io.quarkus.platform:quarkus-bom`,
  `io.quarkus:quarkus-bom`, or `io.quarkus:quarkus-universe-bom`, or an
  inherited parent. The `quarkus-maven-plugin` version alone does not establish
  the application's dependency version. Use Maven's effective POM or
  `dependency:tree` on `io.quarkus:quarkus-core` if necessary.
- Resolve the **effective Java level** of the project: `maven.compiler.release`,
  `maven.compiler.source`/`target`, `<release>`/`<source>` in the compiler
  plugin, toolchains, and also non-POM places that will need to follow
  (`.sdkmanrc`, `.java-version`, `src/main/docker/Dockerfile.*` base images, CI
  workflows). Also note which JDK runs Maven (`mvn -v`); it can differ from the
  target level.
- Identify the reactor root, Maven launcher, test profiles, and any services
  required for validation. Use `./mvnw` instead of `mvn` when the project
  requires its wrapper, retaining the command's remaining arguments.
- Check the Maven version that will run the build (`distributionUrl` in
  `.mvn/wrapper/maven-wrapper.properties`, otherwise `mvn -v`). Newer Quarkus
  lines need newer Maven (Quarkus 2.16 needs 3.8.x, Quarkus 3.x and 4.x need
  3.9.x; confirm in the target version's prerequisites). A Maven that is too old
  fails in confusing ways (plugin mojos that cannot be provisioned, missing
  classes at test startup), so plan to raise the wrapper's Maven version as part
  of the stage that needs it and record it as an agent change.
- Establish the requested destination, if specified. An unspecified destination
  means first completing the current-major stabilization stage. Explain the
  detected Quarkus version, Java level, and immediate stage before executing it.

## 2. Select a single stage

Verify stable releases using official Quarkus release information or Maven
repository metadata (e.g.
`https://repo1.maven.org/maven2/io/quarkus/platform/quarkus-bom/maven-metadata.xml`).
Record the source of the selected version. Do not treat a recipe's name, a
successful exit, or a remembered version as proof that its output is the latest
stable release. Exclude snapshots, alphas, CRs, and other pre-releases when
selecting a stable target.

| Starting state | Immediate stage |
| --- | --- |
| 1.x or older 2.x, project Java level 8 (below 11) | **Stop and ask.** Quarkus 2.x requires Java 11+, and this skill supplies no Java 11 recipe. See "Java 8 projects" below. |
| 1.x or older 2.x, Java 11+ | Upgrade to latest stable 2.x with the Quarkus 1→2 recipe, then validate and pause. |
| Latest stable 2.x, with the move to 3.x confirmed, Java below 17 | Raise the project to Java 17 with the Java 17 recipe, then validate and pause. |
| Latest stable 2.x with the move to 3.x confirmed, or older 3.x — Java 17+ | Upgrade to latest stable 3.x with `quarkus:update`, then validate and pause. |
| Older 3.x, Java below 17 | Raise the project to Java 17 with the Java 17 recipe first, validate and pause, then `quarkus:update`. |
| Latest stable 3.x (3.40.x), with the move to 4.x confirmed, Java below 21 | **Ask which Java level to move to** — see "Java for Quarkus 4" below — then run the matching Java 21 or Java 25 stage, validate and pause. |
| Latest stable 3.x (3.40.x), with the move to 4.x confirmed, Java 21+ | Upgrade to latest stable 4.x with the Quarkus 4 `quarkus:update`, then validate and pause. |
| Already at the requested stable destination | Verify the current state and report that no version change is needed. |

Do not skip a major version: a 1.x application reaches the latest 2.x before
any 3.x work begins, and an older 3.x application reaches the latest 3.40.x
before any 4.x work begins. A request for a distant destination does not remove
the confirmation between completed stages. If starting on the latest stable 2.x
(or 3.x) and the user explicitly requests 3.x (or 4.x), that request authorizes
the first transition (the Java stage when needed); each subsequent stage still
requires the confirmation described in section 4. A request for "the latest
Quarkus" from 3.x therefore ends at 3.40.x and asks before starting 4.x.

**Java for Quarkus 4.** Quarkus 4 requires Java 21 and recommends Java 25; Java
17 is no longer supported. If the project targets a Java level below 21, do not
pick one yourself — the choice affects developer machines, CI, and the runtime
image. Explain this and ask:

1. Java 21 — the new minimum (Java 21 stage below).
2. Java 25 — recommended for Quarkus 4 (Java 25 stage below).
3. Stop here.

If the user's request already names the Java level ("move to Quarkus 4 on Java
25"), that is the answer; do not ask again. If the project is already on Java 21
or newer, no Java stage is needed: continue with the Quarkus 4 stage, and when
it is below 25, mention in the report that Java 25 is recommended.

**Java 8 projects.** Do not run any recipe. Explain that Quarkus 2.x needs Java
11 or newer and that the project currently targets Java 8, then offer the
options and wait for the user's decision:

1. Raise the project to Java 11 following official guidance (JDK 11 migration
   notes, the Quarkus 2.0 migration guide) — the agent can make these changes
   by hand and validate them as their own stage.
2. Go straight to Java 17 with the Java 17 recipe below. Warn that older 1.x
   releases may not build or run on Java 17, so this can leave the application
   broken until the 1→2 stage lands.
3. Stop here.

Use the commands below for each stage. Keep the pinned coordinates, recipe IDs,
plugin versions, and stream; do not silently replace an unavailable recipe or
upgrade the recipe artifacts. If a command cannot resolve, or its target line
conflicts with the verified stable destination (e.g. Maven metadata shows a
newer stable 3.x stream than 3.40, or a newer stable 4.x stream than 4.0),
explain the evidence and ask how to proceed. Do not force an unpublished target
or call an older result "latest."

Run each command from the reactor root with a JDK that satisfies it: JDK 11+
for the 1→2 recipe, JDK 17+ for the Java 17 recipe and for the Quarkus 3
`quarkus:update` (the 3.40 plugin is compiled for Java 17), and JDK 21+ for the
Java 21 stage and the Quarkus 4 `quarkus:update` (JDK 25+ for the Java 25 stage,
so the project can be compiled afterwards). Switching the JDK that runs Maven
(`JAVA_HOME`, SDKMAN) does not change the project's target level; only the
recipes and your recorded changes do.

### Stage: Quarkus 2.x (from 1.x or older 2.x)

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.46.1:run --define rewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-quarkus:2.34.1 --define rewrite.activeRecipes=org.openrewrite.quarkus.quarkus2.Quarkus1to2Migration --define rewrite.exportDatatables=true
```

The recipe moves `quarkus-bom`/`quarkus-universe-bom` and
`quarkus-maven-plugin` to the newest `2.x` (expected: 2.16.12.Final) and applies
the 1→2 API and property migrations. It does not change the Java level.

### Stage: Java 17 (before Quarkus 3.x, when the project targets Java below 17)

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.46.1:run --define rewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-third-party:0.45.0 --define rewrite.activeRecipes=io.quarkus.updates.core.quarkus37.UpgradeToJava17,io.quarkus.updates.core.quarkus37.JavaVersion17 --define rewrite.exportDatatables=true
```

The two recipes split the work: `UpgradeToJava17` updates GitHub Actions
`setup-java`, removes JDK APIs dropped in Java 17, and upgrades the checkstyle
plugin; `JavaVersion17` raises `maven.compiler.release`/`source`/`target` and
the compiler plugin's `<release>` to 17. Neither changes the Quarkus version.
Confirm afterwards that the effective Java level is 17 in every module (a level
set in a parent, profile, or toolchain may need an agent change). Check the
non-POM places recorded in section 1 (Dockerfiles, `.sdkmanrc`, CI) and align
them as agent changes.

### Stage: Quarkus 3.x (latest 3.40 stream)

```bash
mvn -U io.quarkus.platform:quarkus-maven-plugin:3.40.1.2:update -N -Dstream=3.40
```

`quarkus:update` rewrites the platform BOM and plugin versions and applies the
Quarkus update recipes for every intermediate version (javax → jakarta,
RESTEasy Reactive → Quarkus REST renames, config property changes, extension
relocations, Quarkiverse/Camel Quarkus updates). It prints the recipes it
applied; keep that log as evidence. Afterwards confirm the resulting version is
the latest stable 3.40.x in Maven metadata.

The goal first generates the update recipes, then asks `Do you want to apply
the generated update recipes with OpenRewrite? ([y]es, [n]o, [d]ry-run)`. In
batch mode (`-B`) or without a terminal it answers nothing and exits
successfully **without changing any file**, so a green build says nothing about
whether the update ran. Run it without `-B` and answer `y` (e.g. pipe it:
`echo y | mvn -U io.quarkus.platform:quarkus-maven-plugin:3.40.1.2:update -N -Dstream=3.40`),
and before moving on confirm with `git status` that the POM actually changed.
The generated recipe and its log (`target/rewrite/rewrite.yaml`,
`target/rewrite/rewrite.log`) list what was applied; copy them to the temp
directory before the next `clean` deletes them.

### Stage: Java 21 or Java 25 (before Quarkus 4.x, when the project targets Java below 21)

Run only after the user chose the Java level (see "Java for Quarkus 4"). The
recipes are declared in this skill's `assets/` directory (next to this
`SKILL.md`), because they need per-recipe options that cannot be passed on the
command line. Use the absolute path of that directory for
`rewrite.configLocation`; nothing is added to the user's project.

Java 21:

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.46.1:run --define rewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-third-party:0.45.0 --define rewrite.configLocation=<skill-dir>/assets/rewrite-java21.yml --define rewrite.activeRecipes=io.quarkus.upgrade.UpgradeToJava21 --define rewrite.exportDatatables=true
```

Java 25:

```bash
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.46.1:run --define rewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-third-party:0.45.0 --define rewrite.configLocation=<skill-dir>/assets/rewrite-java25.yml --define rewrite.activeRecipes=io.quarkus.upgrade.UpgradeToJava25 --define rewrite.exportDatatables=true
```

Each file runs `io.quarkus.updates.core.quarkus37.SetupJavaUpgradeJavaVersion`
(GitHub Actions `setup-java` versions), `io.quarkus.updates.core.quarkus37.UpgradeJavaVersion`
(`maven.compiler.release`/`source`/`target`), and
`org.openrewrite.maven.UseMavenCompilerPluginReleaseConfiguration` (the compiler
plugin's `<release>`), all set to the chosen version. Neither changes the
Quarkus version. As with the Java 17 stage, confirm the effective Java level in
every module, and align the non-POM places recorded in section 1 (Dockerfile
base images, `.sdkmanrc`, `.java-version`, other CI systems) as agent changes.

### Stage: Quarkus 4.x (latest 4.0 stream)

```bash
mvn -U io.quarkus.platform:quarkus-maven-plugin:4.0.1:update -N -Dstream=4.0
```

It behaves like the Quarkus 3 update — including the confirmation prompt — so
run it as `echo y | mvn -U io.quarkus.platform:quarkus-maven-plugin:4.0.1:update -N -Dstream=4.0`,
confirm with `git status` that the POM changed, keep the generated recipe and
log, and confirm the resulting version is the latest stable 4.0.x in Maven
metadata. The Quarkus 4 update rewrites a lot automatically (for example the
Jackson 2 → 3 dependency switch), but it does not migrate everything: expect
compile errors, test failures, or runtime differences that need the migration
guides (section 3, step 3).

## 3. Inspect and validate the result

After each recipe or update command, complete all these steps before
considering another stage:

1. Inspect the diff and POMs, resolving inherited versions again when needed.
   Confirm the actual resulting Quarkus version (and Java level) against the
   selected stable target across the reactor. If nothing changed unexpectedly,
   or versions are inconsistent (e.g. a module or Quarkiverse extension still on
   the old line), investigate before declaring success.
   - **Record what the recipe changed before you change anything yourself.**
     The final report must separate recipe changes from agent changes (section
     4), and that becomes guesswork once both are mixed in the working tree.
     Right after the command exits, save `git diff --stat`, `git diff`, and
     `git status --porcelain` to the temp directory (new files are untracked and
     do not appear in `git diff`). Also keep the evidence of which sub-recipe
     touched which file: the `Changes have been made to <file> by:` lines in the
     Maven log, the datatables under `target/rewrite/datatables/` when
     `rewrite.exportDatatables=true` was used, and the recipe list printed by
     `quarkus:update`. If the tree had user changes before the command, snapshot
     those first too, so they are attributed to neither.
   - From this point on, keep a short **agent change log**: for every edit you
     make (code, POM, configuration, tests, docs, Dockerfiles, scripts), note the
     file, what changed, and the failure or reason that required it (e.g.
     "compile error in `X.java`", "smoke test: `/q/openapi` 404"). Also log when
     you revert or adjust something the recipe did — that is an agent change too.
2. Run `mvn clean install` from the appropriate reactor root (or its project
   wrapper equivalent) with a JDK that matches the new target level. Do not skip
   tests to manufacture a successful build.
   If resolution warns about an *invalid* or *non-parseable POM* in the local
   repository, or classes are missing that the dependency tree says should be
   present, suspect a corrupted local Maven cache (e.g. an HTML error page
   saved as a `.pom`) or a cached resolution failure rather than the migration:
   retry with `-U`, and if a cached file is corrupt, report it and ask the user
   before deleting anything outside the project.
3. Fix migration-related compilation, unit-test (`@QuarkusTest`), and
   integration-test (`@QuarkusIntegrationTest`/`@NativeImageTest`) failures, and
   rerun the affected validation until it passes. Follow the repository's
   conventions. When the recipe or `quarkus:update` leaves an issue behind — a
   compile error, a failing test, an application that does not start, a failed
   smoke check, an `Unrecognized configuration key` warning, a changed response
   — use the official Quarkus migration guides as the source for the fix:
   - Open the index at https://github.com/quarkusio/quarkus/wiki/Migration-Guides.
     It lists one guide per Quarkus minor version, from 1.x to 4.x.
   - Read the guides for every version between the starting and the resulting
     Quarkus version (e.g. 3.2 → 3.40 means guides 3.3 … 3.40; 3.40 → 4.0 means
     guide 4.0), find the section that explains the issue (renamed packages or
     APIs, removed configuration properties, dropped relocations, changed
     defaults, …), and apply the fix it describes.
   - Cite the guide and section as the reason in the agent change log (e.g.
     "Migration Guide 4.0 → Jackson: `ObjectMapperCustomizer` →
     `JsonMapperBuilderCustomizer`"), so the report shows where every manual fix
     came from.
   - If no guide explains the issue, or the guides cannot be fetched, say so in
     the report and rely on other official Quarkus documentation instead of
     guessing.
4. Check whether integration tests require an explicit profile or command beyond
   `clean install` (e.g. `verify -Dnative`, `-DskipITs=false`); inspect the
   build configuration and documented instructions, and add any such command to
   the checklist. Report skipped or unavailable tests accurately.
5. MANDATORY — run the **entire documented validation checklist** from section 1
   after every stage, and without exception after every major-version upgrade
   (1.x→2.x, 2.x→3.x, 3.x→4.x) and every Java stage (17, 21, 25). This is a hard requirement of this
   skill, not a suggestion; a stage cannot be completed while any item is unrun.
   - Re-read every `README.md`, `AGENTS.md`, and `CLAUDE.md` first: the upgrade
     may have changed what they reference, and you may add items you missed.
   - Execute **every** item — every run approach and every documented
     test/verification command — not a representative subset. Each approach
     exercises different code paths that a major upgrade can break: dev mode
     and Dev Services, the fast-jar vs. über-jar packaging
     (`quarkus.package.type` became `quarkus.package.jar.type` in 3.x),
     native-image reflection, container-image extensions, coverage and analysis
     plugins. One passing approach says nothing about the others.
   - For each run approach: perform its documented setup, start it (background
     long-running processes such as `quarkus:dev` and capture logs), wait for
     the documented readiness signal (e.g. `Listening on: http://…` and
     `/q/health/ready` UP), run the **full smoke test from section 3a** against
     it, then stop it and run its documented cleanup (e.g. Compose `down`)
     before starting the next one, so ports and containers do not collide. Run
     items sequentially. In dev mode, disable interactive input if needed (e.g.
     run with `-Dquarkus.console.enabled=false`) so the process runs unattended.
   - Every other item is also followed by a smoke test (section 3a explains how
     for build-only and test-only items). An item is PASSED only when its own
     command succeeded **and** its smoke test passed.
   - Fix every failure these items surface, then rerun the failing item **and**
     any earlier items the fix could affect, until all pass.
   - A missing prerequisite (e.g. GraalVM/Mandrel for native, Docker for Dev
     Services, a SonarQube server) is not a reason to skip silently. First try to
     satisfy it using the repository's documented setup (e.g. SDKMAN, or a
     container-based native build with `-Dquarkus.native.container-build=true`
     if Docker is available). Steps that need `sudo`, credentials, are
     documented for another OS, or publish results to an external service: ask
     the user before running them, offering the documented command or the
     closest local equivalent. If an item still cannot run, mark it **BLOCKED**
     with the reason; the stage is then not ready and you must not propose the
     next major stage until the user resolves it or explicitly waives that
     specific item.
   - Only if none of these files document any run/validation step, say so
     explicitly, then start the application with an appropriate local command
     (`quarkus:dev` and the packaged `java -jar target/quarkus-app/quarkus-run.jar`,
     or `target/*-runner.jar` on legacy packaging) and still run the full smoke
     test from section 3a.
6. Clean up only processes/resources started for this validation: record the
   PID of every process you launch (and container names/IDs) when you start it,
   and stop exactly those. Never stop processes by name or pattern (`pkill java`,
   `pkill -f quarkus`, `ps | grep quarkus | kill`): the user may have other
   Quarkus or Java processes running, and killing them is destructive. Inspect the
   final diff and retain each checklist item's command, result, and evidence
   (exit code, key log lines, HTTP status, smoke-test results) for reporting.

## 3a. Smoke test after every step and approach

Builds and test suites pass while the running application is broken more often
than one would expect after a Quarkus upgrade: a renamed property is ignored
with only an "Unrecognized configuration key" warning, `/health` moves under
`/q`, Swagger UI disappears outside dev mode, a RESTEasy Reactive change alters
a JSON shape or status code, security defaults change, a native image lacks a
reflection registration. Only exercising the real interfaces of the running
application catches these. That is why every checklist item ends with a smoke
test — no exceptions, including after a fix and its rerun.

**When, per item type:**

| Item type | Smoke test |
| --- | --- |
| Run approach (`quarkus:dev`, `java -jar …quarkus-run.jar`, über-jar, Compose, profile variants) | Run it against that instance while it is up, before stopping it. |
| Build-only (`package`, `-Dnative`, container image build) | Start the produced artifact (via the documented follow-up, e.g. `./target/*-runner`, `docker run`, Compose) and smoke test it. An image or native binary that was built but never started is not validated. |
| Test-only or analysis (`test`, `verify`, `install`, native tests, coverage, Sonar) | Afterwards, start the application with the primary documented run approach and smoke test it. Consecutive test-only items may share one smoke run only if no code, configuration, or dependency changed between them. |

**What to exercise** — every interface the smoke-test plan (section 1) found,
not only what the documentation mentions:

1. **Health:** `/q/health`, `/q/health/live`, `/q/health/ready` (or the
   configured paths) return `UP`, including each documented check.
2. **API docs and Swagger UI:** `/q/openapi` (or the configured path) returns a
   valid document; diff its paths and operations against the baseline when you
   have one. `/q/swagger-ui` returns 200 HTML wherever it is expected to be
   available (dev mode, or always when `always-include` is set), and the
   document it loads is reachable.
3. **REST API:** at least one representative request per resource class /
   route. Assert status **and** content (expected fields, non-empty seed data,
   content type), not only that the call returned 2xx.
4. **Security:** a protected route without credentials returns 401/403; when a
   token helper or test credentials are documented, perform an authenticated
   write (e.g. create → read back → delete/clean up) so you leave no test data
   behind.
5. **Web UI:** if static pages or Qute templates exist, request them and assert
   expected markup/text; load key assets (JS/CSS). In dev mode, also check the
   Dev UI (`/q/dev-ui` on 3.x, `/q/dev` on 2.x) loads. If a headless browser is
   readily available, use it for the main page; otherwise HTTP checks suffice.
6. **Other interfaces** the application exposes (GraphQL, WebSocket, gRPC,
   Kafka/AMQP messaging, scheduler, metrics): one minimal round trip each.
7. **Logs:** scan the startup and request logs for errors, stack traces,
   deprecation warnings, and especially `Unrecognized configuration key`
   warnings — each one is a property the upgrade renamed or removed and must be
   resolved or explained.

Use plain commands (`curl --fail -sS -w '%{http_code}'`, `jq`) so results are
reproducible, and keep a short evidence line per check (URL, status, the
asserted value). Prefer writing the checks once as a small script in a temp
directory and reusing it for every item and every stage — it keeps runs
consistent and makes comparing approaches meaningful. Parameterize the
management prefix, because it can differ between stages (e.g. `/health` on very
old 1.x vs. `/q/health`).

A failed smoke check fails the item: fix the cause, rerun the item and its smoke
test. If a check cannot run (e.g. no credentials for an authenticated call), mark
that check BLOCKED with the reason and ask the user — do not quietly drop it.

## 4. Report and pause

Summarize:

- **Version:** initial → actual resulting Quarkus version and Java level, stage,
  stable-release evidence.
- **Changes:** split into two clearly labeled subsections so the user can tell
  automated migration from hand-made fixes — they carry different risk and
  review needs (recipe edits are mechanical and repeatable; agent edits are
  judgment calls that deserve closer review). Base both on the snapshot and
  change log from section 3, step 1, not on memory. Always include **both**
  headings in every report, even when one side is empty ("None") — a single
  merged change list is not acceptable, because the reader cannot tell which
  edits need closer review.
  - **Changes made by the OpenRewrite recipe / `quarkus:update`:** the exact
    command and recipe ID (with artifact coordinates) or plugin version and
    stream, then the files it modified/added/deleted, grouped by kind of change
    (e.g. platform BOM/plugin version, Java level, dependency coordinates and
    extension renames, `javax.*` → `jakarta.*` imports, renamed properties,
    Panache/RESTEasy API migrations, plugin updates), naming the responsible
    sub-recipe where the log or datatables show it. If it changed nothing, say
    so.
  - **Changes made by the AI agent:** one entry per change with file(s), what
    was changed, and why — the failing build/test/approach/smoke check that
    required it, or another reason (e.g. aligning a Dockerfile base image with
    the new Java level, updating docs/commands). Include any recipe change you
    reverted or adjusted, and say explicitly if you made no manual changes.

  If a single file was touched by both, list it in both subsections, each
  describing only its own part. Pre-existing user changes belong to neither.
  Example layout:

  ```markdown
  ### Changes made by `quarkus:update`
  Command: `mvn -U io.quarkus.platform:quarkus-maven-plugin:3.40.1.2:update -N -Dstream=3.40` — 9 files
  - `pom.xml`: `quarkus.platform.version` 2.16.12.Final → 3.40.x; `quarkus-resteasy-reactive` → `quarkus-rest` (core/3.9 ArtifactRelocations)
  - `src/main/java/**`: `javax.*` → `jakarta.*` imports (core/3alpha JavaxToJakarta)
  - `src/main/resources/application.properties`: `quarkus.package.type` → `quarkus.package.jar.type`

  ### Changes made by the AI agent
  | File | Change | Reason |
  | --- | --- | --- |
  | `src/main/docker/Dockerfile.jvm` | Base image `openjdk-11` → `openjdk-17` | Java 17 stage: container smoke test failed to start |
  | `GreetingResourceTest.java` | Replaced removed `@NativeImageTest` with `@QuarkusIntegrationTest` | `mvn clean install` compile error |
  ```
- **Validation:** the full documented validation checklist as a table, one row
  per item: source (file → heading), command(s), result (PASSED / FAILED /
  BLOCKED / WAIVED by user), smoke-test result, and evidence. Every item found in
  section 1 must appear; do not drop rows that were not run.
- **Smoke tests:** the interfaces exercised (health, OpenAPI, Swagger UI, REST
  endpoints, auth, web UI, Dev UI, others) with URL and result, which items each
  smoke run covered, and any differences from the pre-upgrade baseline.
- **Status:** ready only when every checklist item and its smoke test PASSED (or
  the user explicitly waived it); otherwise describe the incomplete stage
  without implying success.
- **Next:** only after every checklist item passed or was waived, ask whether
  the user wants additional validation. If a further stage is still pending
  (Java 17, Quarkus 3.x, Java 21/25, Quarkus 4.x), ask whether to proceed to
  that specific next stage.
  Wait for the answer before executing it.

Keep the final report concise and distinguish executed checks from suggested or
blocked ones. For a planning-only request, give a plan and do not claim execution.
