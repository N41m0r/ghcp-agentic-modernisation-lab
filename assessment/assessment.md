# Assessment Report - Order Service Java and Spring Boot Migration

**Module:** `app\Java - Spring Boot\Order Service`  
**Assessment date:** 2026-09-17  
**Scope:** assessment only; no module source or Maven manifest was changed.

## Executive summary

The service is a small Maven Spring Boot REST API using MVC, JPA, Bean Validation, H2, and tests. It is currently compiled for Java 8 on Spring Boot 2.7.18. The target of Java 25 and Spring Boot 3.5.x is feasible, but it has three compile-time migration blockers: the Java 8 compiler settings, `javax.*` persistence/validation imports, and the removed `WebMvcConfigurerAdapter` base class. Two explicit dependencies are knowingly vulnerable. Upgrade the security dependencies first, establish a Java 17 baseline, make the Boot 3/Jakarta change, then advance the JDK in controlled steps to Java 25.

## Current and target state

| Area | Current evidence | Target state |
| --- | --- | --- |
| JDK and compiler | `pom.xml` sets `java.version`, `maven.compiler.source`, and `maven.compiler.target` to `8`; the explicit `maven-compiler-plugin:3.15.0` configuration also sets source/target 8 | Compile and test on Java 25, preferably with `<release>25</release>` rather than separate source/target settings |
| Framework | `spring-boot-starter-parent:2.7.18` | Spring Boot 3.5.x (requires Java 17 or later) |
| API namespace | `javax.persistence.*` and `javax.validation.*` imports | Jakarta EE 9+ `jakarta.persistence.*` and `jakarta.validation.*` imports |
| MVC configuration | `WebConfig` extends `WebMvcConfigurerAdapter` | `WebConfig` implements `WebMvcConfigurer` |
| Build plugins | Compiler plugin is explicit; Boot Maven plugin version is inherited from the Boot 2.7.18 parent | Retain a Java-25-capable compiler plugin configuration and inherit `spring-boot-maven-plugin` from the Boot 3.5.x parent |
| Explicit security dependencies | `log4j-core:2.14.1`, `commons-text:1.9` | Upgrade to supported patched versions; do not retain these overrides at vulnerable versions |

## Compatibility blockers and concrete findings

1. **Jakarta namespace migration is mandatory.** `src\main\java\com\contoso\demo\orderservice\model\Order.java` imports JPA and validation types from `javax.persistence` and `javax.validation.constraints`; `web\OrderController.java` imports `javax.validation.Valid`. Spring Boot 3 uses Spring Framework 6 and Jakarta EE APIs, so these imports must become their `jakarta.*` equivalents. The entity and controller tests must be recompiled and run after the rewrite.

2. **`WebMvcConfigurerAdapter` was removed.** `config\WebConfig.java` imports and extends `org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter`. It is not available in Spring Framework 6. Replace the inheritance with `implements WebMvcConfigurer`; its default interface methods preserve the adapter's purpose.

3. **All Java 8 compiler settings block the target.** The parent property settings and the plugin's duplicate `<source>8</source>/<target>8</target>` must be made consistent with the selected JDK. Use `--release`/`<release>` to prevent accidental compilation against APIs newer than the intended release. Spring Boot 3.5.x cannot run or compile on Java 8; Java 17 is its minimum baseline.

4. **The Spring Boot parent upgrade changes major runtime libraries.** Boot 3.5 moves to Spring Framework 6, Tomcat 10.1, Hibernate ORM 6, and Jakarta APIs. The application has no direct Hibernate API imports beyond JPA annotations, but JPA schema generation and H2-backed repository behavior need regression testing because Hibernate 6 is a major upgrade.

5. **Java 8-to-25 source scan found no additional direct removed-JDK API blocker.** The eight main Java files and four test files contain no `SecurityManager`, `setSecurityManager`, `finalize`, JAXB (`javax.xml.bind`), applet, or RMI Activation usage. Java 17/21/25 language features are not required to migrate this source; the material risk is framework/library compatibility and test tooling supplied by the new Boot BOM.

6. **Configuration review.** `application.properties` uses an H2 in-memory JDBC URL, `spring.jpa.hibernate.ddl-auto=update`, and the H2 console. No property-key rename was identified, but `ddl-auto=update` should be exercised against Hibernate 6 and the H2 console should remain restricted to non-production profiles.

## CVE findings

The first four items are directly declared in `pom.xml`. The final three are reachable from the web/JPA starter graph managed by the old Boot parent; exact resolved versions should be captured with `mvn dependency:tree` before remediation because Maven dependency management may be overridden by an environment or corporate parent. Advisory ranges and fixed versions were checked against GitHub Security Advisories where available and NVD references otherwise.

| Dependency and version | CVE | Affected range covering the current version | Minimum fixed version | Evidence |
| --- | --- | --- | --- | --- |
| `org.apache.logging.log4j:log4j-core:2.14.1` | CVE-2021-44228 | Java 8 Log4j 2.x releases through 2.14.1 | 2.15.0; use 2.17.1 or later to also address follow-on CVEs | [GHSA-jfh8-c2jp-5v3q](https://github.com/advisories/GHSA-jfh8-c2jp-5v3q), [NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-44228) |
| `org.apache.logging.log4j:log4j-core:2.14.1` | CVE-2021-45046 | Java 8 Log4j 2.x releases through 2.15.0 | 2.16.0; use 2.17.1 or later | [GHSA-7rjr-3q55-vv33](https://github.com/advisories/GHSA-7rjr-3q55-vv33), [NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-45046) |
| `org.apache.logging.log4j:log4j-core:2.14.1` | CVE-2021-44832 | 2.0-beta7 through 2.17.0 (requires attacker control of Log4j configuration and JDBC Appender conditions) | 2.17.1 | [GHSA-8489-44mv-ggj8](https://github.com/advisories/GHSA-8489-44mv-ggj8), [NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-44832) |
| `org.apache.commons:commons-text:1.9` | CVE-2022-42889 | 1.5 through 1.9 | 1.10.0 | [GHSA-599f-7c49-w659](https://github.com/advisories/GHSA-599f-7c49-w659), [NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-42889) |
| Spring Framework 5.3.31 (`spring-web` via `spring-boot-starter-web`) | CVE-2024-22243 | 5.3.0 through 5.3.32 | 5.3.33 | [Spring advisory](https://spring.io/security/cve-2024-22243), [GHSA-ccgv-vj62-xf9h](https://github.com/advisories/GHSA-ccgv-vj62-xf9h) |
| Tomcat 9.0.83 (`tomcat-embed-core` via web starter) | CVE-2024-24549 | 9.0.0-M1 through 9.0.85 | 9.0.86 | [NVD](https://nvd.nist.gov/vuln/detail/CVE-2024-24549) |
| SnakeYAML 1.30 (managed by Boot 2.7.18; verify resolution) | CVE-2022-1471 | 1.x through 1.33 | 2.0 | [GHSA-mjmj-j48q-9wg2](https://github.com/advisories/GHSA-mjmj-j48q-9wg2), [NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-1471) |

The Log4j and Commons Text declarations are explicit overrides and therefore must be deliberately upgraded or removed. A Boot 3.5.x parent updates its managed Spring, Tomcat, Hibernate, Jackson, and SnakeYAML generations, but it does not make an explicit vulnerable dependency declaration safe by itself.

## Recommended staged migration sequence

1. **Establish a baseline and remediate explicit CVEs.** Record the current test results and resolved dependency tree. Upgrade `log4j-core` to at least 2.17.1 and `commons-text` to at least 1.10.0 (prefer current compatible releases), then test. This isolates urgent security work from the framework namespace change.

2. **Move the build and CI runtime to Java 17.** Update the JDK, compiler properties, and compiler-plugin configuration together; use `<release>17</release>`. Java 17 is the Boot 3 floor and provides a stable intermediate point for diagnosing JDK/toolchain issues independently of Spring changes.

3. **Upgrade to Spring Boot 3.5.x and complete the Jakarta migration as one change.** Bump the Boot parent, replace `javax.*` imports with `jakarta.*`, replace `WebMvcConfigurerAdapter`, and remove any obsolete explicit version overrides that conflict with the Boot 3 BOM. Run unit, MVC, JPA/H2, and startup tests. Pay particular attention to Hibernate 6 DDL and repository behavior.

4. **Advance from Java 17 to Java 21.** Update the release target and CI image, then rerun the full backend test suite. This LTS checkpoint limits the size of any JDK-specific troubleshooting step.

5. **Advance from Java 21 to Java 25.** Update the release target/toolchain and validate build, tests, application startup, H2 schema creation, request validation, and web configuration. Keep the Boot 3.5.x BOM/plugin pair intact so test libraries such as Mockito and Byte Buddy remain aligned with the JDK.

## Assessment boundaries

The React/Vite frontend is out of scope. No source, dependency manifest, or application configuration was modified during this assessment. The report is the only artifact created.
