# Validation Summary — Java 8 → Java 21 Upgrade

## JDK Versions

| | Source | Target |
|---|---|---|
| Root module (`movie-service`) | Java 1.8 | Java 21 |
| Sibling module (`movie-service-utils`) | Java 17 | Java 21 |

The two modules previously targeted different JDKs (1.8 vs 17); both now target Java 21 and are reconciled.

## Build-File and Dependency Changes (Before / After)

### `pom.xml` (movie-service)

| Item | Before | After |
|---|---|---|
| `java.version` | `1.8` | `21` |
| `maven.compiler.release` | (absent, used source/target) | `21` |
| spring-boot-starter-parent | 2.0.5.RELEASE | 3.2.12 |
| maven-compiler-plugin | 3.8.1 (`source`/`target`) | 3.13.0 (`release`) |
| software.amazon.awssdk:bom | 2.14.27 | 2.29.52 |
| com.amazonaws.samples:movie-service-utils | 0.1.0 | 0.3.0 |
| org.json:json | 20200518 | 20240303 |
| log4j-api / log4j-core | 2.13.3 | 2.25.4 |
| junit:junit | 4.13.1 | 4.13.2 |
| org.junit.vintage:junit-vintage-engine | (absent) | added (managed by Spring Boot BOM) |
| org.mockito:mockito-all | 1.10.19 | replaced by org.mockito:mockito-core (Spring Boot 3.2.12-managed, 5.x) |
| javax.validation:validation-api | 2.0.1.Final | replaced by jakarta.validation:jakarta.validation-api 3.0.2 |

### `movie-service-utils/pom.xml`

| Item | Before | After |
|---|---|---|
| module version | 0.2.0 | 0.3.0 |
| `java.version` / compiler source / target / release | 17 | 21 |
| maven-compiler-plugin | 3.13.0 | 3.13.0 (unchanged) |

### `build.gradle`

| Item | Before | After |
|---|---|---|
| spring-boot-gradle-plugin | 2.3.0.RELEASE | 3.2.12 (aligned with pom.xml) |
| source/targetCompatibility | 1.8 | 21 |
| awssdk bom | 2.14.27 | 2.29.52 |
| org.json:json | 20200518 | 20240303 |
| junit | 4.11 | 4.13.2 |
| junit-vintage-engine | (absent) | added as `testRuntimeOnly` |
| configurations | `compile` / `testCompile` (removed in Gradle 7+) | `implementation` / `testImplementation` |

### `Dockerfile`

| Item | Before | After |
|---|---|---|
| base image (build + runtime) | `maven:3.6.1-amazoncorretto-8` | `maven:3.9-amazoncorretto-21` |
| `MAVEN_VERSION` | 3.6.1 | 3.9.9 |

### `.gitlab-ci.yml`

| Item | Before | After |
|---|---|---|
| `BUILD_IMAGE` | `maven:3.8-openjdk-8` | `maven:3.9-amazoncorretto-21` |
| `detect-java-version` check | branched on Java 8 vs 17 in pom.xml | asserts `<java.version>21</java.version>`, pins `maven:3.9-amazoncorretto-21` |
| Dockerfile rewriting in `package` stage | conditionally generated a Java-17 Dockerfile | removed; the single Java 21 Dockerfile is used directly |
| `q-code-transformation` rule | excluded when `BUILD_IMAGE !~ /openjdk-17/` | JDK-based exclusion removed (image is always JDK 21) |

## Source Files Modified

| File | Why |
|---|---|
| `src/main/java/.../utils/Security.java` | `javax.security.cert` was removed in Java 15. Rewritten to use `java.security.cert.X509Certificate`, obtained via `java.security.cert.CertificateFactory.getInstance("X.509").generateCertificate(InputStream)`, with `java.security.cert` exception types. Method signature and return type (`Certificate`) preserved. |
| `src/main/java/.../movies/MoviesController.java` | `javax.validation.Valid` → `jakarta.validation.Valid` (Jakarta EE 9+ namespace required by Spring Boot 3). Also updated the `MovieUtils` call to the API provided by movie-service-utils 0.3.0 (`isValidMovie(name, movieId)`), since the 0.1.0 method `isValidMovieName` no longer exists in the upgraded dependency. |
| `src/test/java/.../movies/MockTest.java` | `org.mockito.Matchers` was removed in Mockito 2+; switched to `org.mockito.ArgumentMatchers`. |
| `src/test/java/.../movies/MoviesControllerTest.java` | `MockitoAnnotations.initMocks` removed in Mockito 5; switched to `MockitoAnnotations.openMocks`. |

No tests were removed or disabled; all four test classes remain enabled.

## Build and Test Commands Executed

Toolchain used: Apache Maven 3.9.9, Amazon Corretto 21.0.11.

1. `mvn install:install-file -Dfile=./movie-service-utils/built-library/0_3_0/movie-service-utils-0.3.0.jar -DgroupId=com.amazonaws.samples -DartifactId=movie-service-utils -Dversion=0.3.0 -Dpackaging=jar` — **SUCCESS** (the root module consumes the utils library as a prebuilt jar, as the GitLab pipeline also does).
2. `mvn clean verify` (root `movie-service`) — **BUILD SUCCESS**. Tests: `Tests run: 7, Failures: 0, Errors: 0, Skipped: 0` across `MathTest`, `MovieTest`, `MockTest`, `MoviesControllerTest`. Boot jar `target/movie-service-0.1.0.jar` repackaged by spring-boot:3.2.12.
3. `mvn clean install` (in `movie-service-utils`) — **BUILD SUCCESS** at release 21.

### Failures encountered and how they were resolved

- **Compilation failure on `javax.security.cert`** — the package does not exist in JDK 21. Resolved by migrating `Security.java` to `java.security.cert` with `CertificateFactory`.
- **Compilation failure on `javax.validation.Valid`** — Spring Boot 3 ships Jakarta EE 9+. Resolved by migrating to `jakarta.validation` and the `jakarta.validation-api:3.0.2` dependency.
- **Compilation failure on `MovieUtils.isValidMovieName`** — not present in movie-service-utils 0.3.0. Resolved by calling `isValidMovie(movieName, movieId)`.
- **Test compilation failures** (`org.mockito.Matchers`, `MockitoAnnotations.initMocks`) after Mockito moved to 5.x under Spring Boot 3.2.12. Resolved with `ArgumentMatchers` and `openMocks`.
- **JUnit 4 tests not discovered** by the Surefire/JUnit 5 platform of Spring Boot 3. Resolved by adding `org.junit.vintage:junit-vintage-engine` (Maven and Gradle), keeping existing JUnit 4 tests intact.

No build or test step is currently failing.

## Changes Not Completed

- **`.github/workflows/q-code-transformation.yml` was intentionally left untouched** (and a prior in-progress edit to it was reverted) per the explicit constraint: the git credentials in use lack the GitHub `workflow` scope, so any change under `.github/workflows/` causes the whole push to be rejected. That workflow still pins JDK 8/17 — see Manual Follow-Ups.
- **Gradle build was not executed.** `build.gradle` was updated for parity with `pom.xml` (Spring Boot 3.2.12, Java 21), but the repository contains no Gradle wrapper and Gradle is not installed in this environment, so the Gradle build could not be verified. Maven is the build path exercised by the Dockerfile and the GitLab pipeline.
- **Gradle wrapper was not upgraded to 8.11+** for the same reason: no `gradlew`/`gradle/wrapper` files exist in the repository, and adding a wrapper is outside the scope of this upgrade.
- Groups of the transformation definition covering database/ORM, connection pooling, Swagger/SpringDoc, caching, and Apache Commons/HTTP libraries are **not applicable** — none of those dependencies exist in this project.

## Manual Follow-Ups

1. **GitHub Actions JDK pin (required).** `.github/workflows/q-code-transformation.yml` still uses `actions/setup-java` with `java-version: 8` (or `17` post-transformation) and passes `-Djava.version=1.8`/`17`. These values no longer match the project (Java 21) and the workflow will fail. A maintainer with the GitHub `workflow` credential scope must update that file to JDK 21 (`java-version: '21'`, `-Djava.version=21`) and refresh the Java 8/17 install steps and echo statements. It was deliberately not modified here.
2. **Verify the Gradle build** in an environment with Gradle 8.11+ (or add a Gradle wrapper at 8.11+), since `build.gradle` changes are unverified.
3. **Rebuild and re-publish `movie-service-utils-0.3.0.jar`** from the Java 21 sources into `movie-service-utils/built-library/0_3_0/` in your artifact flow if your pipeline requires a byte-identical, JDK-21-compiled artifact.
4. **Runtime smoke test against AWS AppConfig** (the application calls the AppConfig API with AWS SDK 2.29.52); only unit tests were executed here.
