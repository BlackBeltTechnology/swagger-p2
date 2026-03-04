# Swagger P2 Site - Project Documentation

## Project Overview

**Repository:** BlackBeltTechnology/swagger-p2
**License:** Eclipse Public License 2.0 (EPL-2.0)
**Java Version:** 21
**Build System:** Maven 3.9.4 with Maven Wrapper (`./mvnw`)

1. Builds an Eclipse P2 repository (update site) that wraps `io.swagger:swagger-core:1.5.24` and its transitive dependencies as OSGi bundles
2. Generates P2 metadata (`content.xml`, `artifacts.xml`) so Eclipse-based tools can install Swagger via standard Update Site mechanism
3. Packages the P2 repository as a distributable ZIP archive
4. Deploys to JUDO Nexus (internal), Maven Central (Sonatype), and JUDO Nexus P2 site via WebDAV

## Directory Structure

```
swagger-p2/
├── pom.xml                    # Root POM — build config, plugins, profiles, version
├── mvnw / mvnw.cmd            # Maven Wrapper scripts (Unix / Windows)
├── .mvn/
│   ├── extensions.xml         # Wagon extensions (file, WebDAV) for deployment
│   ├── jvm.config             # JVM heap settings: -Xms1024m -Xmx2048m
│   └── wrapper/               # Maven Wrapper JAR and properties
├── src/
│   └── assembly/
│       └── assembly.xml       # Assembly descriptor for ZIP packaging
├── .github/
│   ├── workflows/             # GitHub Actions CI/CD pipelines
│   ├── CIFLOW.md              # CI/CD workflow documentation with diagrams
│   ├── ISSUE_TEMPLATE/        # Bug report, feature request templates
│   ├── dependabot.yml         # Automated dependency update config (Maven, daily)
│   └── labels.yml             # GitHub label definitions
├── openspec/                  # OpenSpec configuration and specs
│   ├── config.yaml
│   ├── specs/
│   └── changes/
├── .vscode/settings.json      # VS Code Java settings
├── .zed/settings.json         # Zed editor Java settings
├── README.md                  # Project introduction and quick start
├── CONTRIBUTING.md            # Development guide and submission guidelines
└── LICENSE.txt                # EPL-2.0 full license text
```

## Core Modules

This is a **single-module packaging project** — there are no Maven submodules and no Java source code.

| Component | Type | Purpose |
|-----------|------|---------|
| `pom.xml` | Build config | Declares `swagger-core:1.5.24` as the artifact to wrap, configures p2-maven-plugin and assembly |
| `src/assembly/assembly.xml` | Assembly descriptor | Defines how `target/repository/` is packaged into a ZIP with `-site` classifier |
| `.mvn/extensions.xml` | Maven extension | Loads `wagon-file` and `wagon-webdav-jackrabbit` for deployment transport |

## Technology Stack

### Core Technologies
- **Eclipse P2** — provisioning platform for OSGi bundle distribution
- **p2-maven-plugin 2.0.0** — generates P2 repository metadata from Maven artifacts
- **maven-assembly-plugin 3.4.2** — packages repository as distributable ZIP
- **flatten-maven-plugin 1.3.0** — resolves CI-friendly `${revision}` version property

### Build & Quality
- **Maven 3.9.4** via Maven Wrapper
- **Java 21** (Zulu distribution)
- **sign-maven-plugin 1.1.0** — GPG artifact signing
- **wagon-webdav-jackrabbit 3.5.2** — WebDAV transport for P2 site upload
- **nexus-staging-maven-plugin** — Maven Central release staging

## Build Commands

```bash
# Full build — generates P2 repository and ZIP
./mvnw clean install

# Build without tests (functionally same for this project)
./mvnw clean install -DskipTests

# Package only (no install to local repo)
./mvnw clean package

# Deploy snapshot to JUDO Nexus
./mvnw clean deploy -Prelease-judong

# Deploy to Maven Central
./mvnw clean deploy -Prelease-central,sign-artifacts

# Upload P2 site to JUDO Nexus P2 repository
./mvnw clean deploy -Prelease-p2-judong

# Test deployment to local /tmp
./mvnw clean deploy -Prelease-dummy

# Check effective version
./mvnw help:evaluate -Dexpression=project.version -q -DforceStdout
```

### Maven Profiles

| Profile | Purpose |
|---------|---------|
| `sign-artifacts` | GPG-sign all artifacts before deployment |
| `release-dummy` | Deploy to local `/tmp` directory for testing |
| `release-judong` | Deploy to JUDO Nexus (snapshots and releases) |
| `release-central` | Deploy to Maven Central via Sonatype OSSRH |
| `release-p2-judong` | Upload P2 site to JUDO Nexus P2 repository via WebDAV |
| `generate-github-asciidoc-diagrams` | Generate PNG CI/CD diagrams from AsciiDoc |
| `update-source-code-license` | Apply EPL-2.0 headers to source files |

## Key Configuration Files

| File | Purpose |
|------|---------|
| `pom.xml` | Main build configuration: plugins, profiles, deployment targets, version |
| `.mvn/extensions.xml` | Wagon extensions for file and WebDAV transport |
| `.mvn/jvm.config` | JVM memory settings (`-Xms1024m -Xmx2048m`) |
| `.mvn/wrapper/maven-wrapper.properties` | Maven version and download URL |
| `src/assembly/assembly.xml` | ZIP packaging descriptor for P2 site |
| `.github/workflows/build.yml` | Main CI/CD build pipeline |
| `.github/workflows/release.yml` | Manual release workflow |
| `.github/dependabot.yml` | Automated Maven dependency updates (daily) |

## Development Environment

**Required:**
- Java 21 JDK (Zulu distribution recommended)
- Maven 3.9.4+ (or use `./mvnw` wrapper which downloads it automatically)
- Git

**For deployment (optional):**
- GPG keys configured for artifact signing
- JUDO Nexus credentials in `~/.m2/settings.xml` (server ID: `judong-nexus-distribution`)
- Sonatype OSSRH credentials in `~/.m2/settings.xml` (server ID: `ossrh`)

## Git Workflow

- **Main Branch:** `develop`
- **Versioning:** CI-friendly `${revision}` property, currently `1.0.4-SNAPSHOT`
- **Branching model:** GitFlow — `develop`, `feature/JNG-*`, `release/*`, `bugfix/JNG-*`, `support/JNG-*`, `hotfix/JNG-*`, `master`
- **Commit convention:** Every commit must include a JIRA ticket number (`JNG-XXX`)
- **CI/CD:** GitHub Actions on `judong` custom runner; see [CIFLOW.md](.github/CIFLOW.md) for detailed workflow diagrams

## Important Notes

1. This project has **no Java source code** — it is purely a build/packaging project that wraps a Maven artifact as an Eclipse P2 site
2. The `${revision}` property in `pom.xml` is the single source of truth for the project version; the `flatten-maven-plugin` resolves it during build
3. The `p2-maven-plugin` runs during the `package` phase and generates the P2 site at `target/repository/`
4. WebDAV deployment (profile `release-p2-judong`) uploads to a versioned path: `p2-judong/swagger-p2/<version>/`
5. Maven Central deployment requires both `release-central` and `sign-artifacts` profiles to be active
6. Dependabot is configured to check Maven plugin updates daily but ignores `guava`, `slf4j-api`, and `jruby-complete`

## Related Documentation

- [README.md](README.md) — Project introduction and quick start
- [CONTRIBUTING.md](CONTRIBUTING.md) — Development environment, branching, and submission guidelines
- [.github/CIFLOW.md](.github/CIFLOW.md) — CI/CD workflow documentation with Mermaid diagrams
- [JUDO Community CONTRIBUTING](https://github.com/BlackBeltTechnology/judo-community/blob/develop/CONTRIBUTING.adoc) — Parent project contribution guide
