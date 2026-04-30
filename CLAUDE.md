# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Jasig Central Authentication Service (CAS) Server, version 3.4.12-SNAPSHOT — a Spring/Spring-Webflow-based SSO server. This is the legacy 3.4.x line: `groupId=org.jasig.cas`, parent `org.jasig.parent:jasig-parent:21`, targeting **Java 1.5** source/target and J2EE 1.3 (Servlet API 2.5). The codebase predates Apereo's rename and many of its declared repositories/dependencies are unobtainable from modern Maven Central.

## Build commands

Use Maven from the repo root. The pom enforces JDK >=1.5 and Maven >=2.0.9, but Maven 3.x with JDK 8 is what works in practice.

- `mvn -B package` — full reactor build, produces JARs and the WAR (`cas-server-webapp/target/*.war`).
- `mvn -pl cas-server-core -am test` — build and test a single module with its dependencies.
- `mvn -pl cas-server-core -Dtest=SomeClassTests test` — run a single test class.
- `mvn -DskipTests package` — skip tests.
- Surefire only picks up `**/*Tests.java` and excludes `Abstract*.java` (configured in root POM).

### Network-dependent tests

A handful of tests reach out over HTTP and will fail offline (or whenever the legacy hosts they hit are dead, which is most of the time now). The known offenders both target `http://www.jasig.org` (defunct):

- `cas-server-core` → `org.jasig.cas.util.HttpClientTests`
- `cas-server-core` → `org.jasig.cas.authentication.handler.support.HttpBasedServiceCredentialsAuthenticationHandlerTests`

Skip with `-Dtest='!HttpClientTests,!HttpBasedServiceCredentialsAuthenticationHandlerTests'` if needed, or use `-DskipTests` for a faster local build.

### Continuous integration

CI runs on **GitHub Actions** via `.github/workflows/build.yml` (push, pull_request, manual dispatch). The workflow uses Zulu JDK 8, Maven dependency caching, and invokes `mvn -B -U -e -fae -s .github/maven-settings.xml package`. It uploads built WAR/JAR artifacts and Surefire reports.

The Maven step is currently `continue-on-error: true` — see the next section for why. Workflow status will go green even when `mvn package` fails; the job log still shows the Maven failure. Once the missing legacy deps are addressed, drop `continue-on-error` so CI reflects real build status.

### CI build environment quirks

GitHub Actions builds via `.github/workflows/build.yml` and **must** pass `-s .github/maven-settings.xml`. That settings file mirrors `external:http:*` repositories through HTTPS Maven Central because Maven 3.8.1+ blocks the POM's HTTP repos (`http://developer.ja-sig.org/maven2`, `http://repository.jboss.org/...`). It also activates a profile that adds `https://build.shibboleth.net/nexus/content/repositories/releases/` for `org.opensaml:opensaml:1.1b`. **Do not delete `.github/maven-settings.xml` without also fixing the POM**, or CI will instantly fail at parent-POM resolution.

### Known broken: missing legacy artifacts

`mvn package` currently fails inside `cas-server-core` because two compile-scope dependencies are no longer obtainable from any free public Maven repository:

| GAV | Why it's gone | Possible fixes |
| --- | --- | --- |
| `org.opensaml:opensaml:1.1b` | Pre-Apereo OpenSAML 1.x; historically on Shibboleth's Nexus, but the `1.1b` artifact may not be there. The CI settings file already adds the Shibboleth releases repo. | Vendor the jar via `mvn install:install-file`; or upgrade to OpenSAML 2.x (API change). |
| `javax.xml:xmldsig:1.0` | Sun-licensed JCP RI; was never redistributable to Central. The class set is built into the JDK from Java 6 onward (`java.xml.crypto`). | Change scope to `provided` since the JDK ships it; or vendor the jar. |

The plugin repo `https://nexus.codehaus.org/...` declared in the root POM is also DNS-dead — Maven retries it on every artifact and falls through to Central, producing noisy warnings but not failing.

Until both artifacts are resolved, the workflow's `Build with Maven` step is marked `continue-on-error: true`. **Once they are fixed, remove `continue-on-error` from `.github/workflows/build.yml` so CI reflects real status.**

## Module architecture

The reactor (defined in the root `pom.xml`) is layered and the build order matters:

1. **`cas-server-core`** — protocol implementation. Contains `CentralAuthenticationServiceImpl` (the central façade), and the `authentication`, `ticket`, `services`, `validation`, `web`, `audit`, `aspect`, `remoting`, and `util` packages. Everything else depends on this.
2. **Authentication-handler / attribute-source modules** — `cas-server-support-{generic,jdbc,ldap,legacy,openid,radius,spnego,trusted,x509}`. Each plugs an `AuthenticationHandler` and/or `PrincipalResolver` into the core via Spring wiring.
3. **Ticket-registry / cache integrations** — `cas-server-integration-{berkeleydb,jboss,memcached,restlet}`. Alternative `TicketRegistry` implementations.
4. **`cas-server-webapp`** — the actual deployable WAR. Spring-Webflow login flow, JSP views under `src/main/webapp/WEB-INF/view/jsp/default/ui`, themes under `src/main/webapp/themes/`. Skin customization is the typical deployer entry point (see `INSTALL.txt`).
5. **`cas-server-uber-webapp`** — packaging-only WAR pulling in every support/integration module; convenience artifact, not the canonical deployable.
6. **`cas-server-compatibility`** — interop tests against the CAS protocol.
7. **`cas-server-documentation`** — docs-only POM module.

Cross-cutting concerns (audit logging via Inspektr, perf timing) are woven via AspectJ — the root POM binds `aspectj-maven-plugin:compile` into every module, so adding `@Aspect` classes anywhere is picked up automatically.

## Adding a new authentication source

The dominant extension point. Each `cas-server-support-{generic,jdbc,ldap,...}` module follows the same pattern, so to add support for a new credential type:

1. Create a new module `cas-server-support-<name>` (clone the structure of `cas-server-support-jdbc`) and add it to the reactor in the root `pom.xml`.
2. Implement `org.jasig.cas.authentication.handler.AuthenticationHandler` (often by extending `AbstractAuthenticationHandler` or a credential-typed base such as `AbstractUsernamePasswordAuthenticationHandler`).
3. If the handler returns user attributes, also implement `org.jasig.cas.authentication.principal.CredentialsToPrincipalResolver`.
4. Wire the handler into `cas-server-webapp/src/main/webapp/WEB-INF/deployerConfigContext.xml` under the `authenticationManager` bean's `authenticationHandlers` list (and `credentialsToPrincipalResolvers` if applicable). The default `SimpleTestUsernamePasswordAuthenticationHandler` declared there is the placeholder deployers replace.
5. Tests live under the new module's `src/test/java/.../*Tests.java`.

## Conventions

- Test class naming: `*Tests.java` (not `*Test`). Abstract base test classes use the `Abstract` prefix and are excluded from Surefire.
- Spring XML wiring under `src/main/resources/` and `src/main/webapp/WEB-INF/` is the source of truth for how modules compose; pure Java introspection will miss the actual graph.
- Source/target is locked to 1.5 in the root POM; do not introduce language features beyond Java 5 (no generics-of-generics inference, no try-with-resources, no diamond operator).
