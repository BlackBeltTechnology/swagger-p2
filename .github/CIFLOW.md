# Development Version and Branch Handling

This document describes the CI/CD pipeline, branching strategy, and version management for the swagger-p2 project.

## Branches

The versioning policy follows [GitFlow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow). Each branch type serves a specific role in the development lifecycle:

| Branch | Purpose | Base branch |
|--------|---------|-------------|
| `develop` | Latest development sources of the current active version | — |
| `feature/JNG-NNN_summary` | New features for the next release | `develop` |
| `release/X.Y` (or `X_Y_betaN`) | Release stabilization and testing | `develop` |
| `bugfix/JNG-NNN_summary` | Fixes applied during release testing | `release/*` |
| `support/JNG-NNN_summary` | Minor updates to a previous release | `release/*` |
| `hotfix/JNG-NNN_summary` | Critical fixes for production | `master` |
| `master` | Latest released sources | — |

### Branch Flow

```mermaid
gitGraph
    commit id: "init"
    branch develop order: 1
    commit id: "dev-1"
    branch feature/JNG-1 order: 2
    commit id: "feat-1a"
    commit id: "feat-1b"
    checkout develop
    merge feature/JNG-1 id: "merge-feat-1"
    branch feature/JNG-3 order: 3
    commit id: "feat-3"
    checkout develop
    merge feature/JNG-3 id: "merge-feat-3"
    branch release/1.0-beta1 order: 4
    commit id: "stabilize"
    branch bugfix/JNG-4 order: 5
    commit id: "fix-4"
    checkout release/1.0-beta1
    merge bugfix/JNG-4 id: "merge-fix"
    checkout main
    merge release/1.0-beta1 id: "v1.0.3"
    checkout develop
    merge release/1.0-beta1 id: "back-merge"
    commit id: "next-dev"
```

## Version Numbers

Versions follow **semantic versioning** with specific rules for each branch type:

| Event | Version action |
|-------|---------------|
| Create `feature/*` branch | No version change |
| Start `release/*` branch | Increment 2nd number on `develop` |
| Create `bugfix/*` branch | No version change (applied on release branch) |
| Create `support/*` branch | Increment 3rd number |
| Create `hotfix/*` branch | Increment 4th number |

## GitHub Actions Workflows

The CI/CD pipeline consists of several interconnected GitHub Actions workflows running on the `judong` custom runner with Java 21 (Zulu).

### build.yml — Main Build Pipeline

**Triggers:** Push to `develop`, pull requests to `develop`, `master`, `increment/*`, or `release/*`

```mermaid
flowchart TD
    A[Push / PR event] --> B{Base branch?}
    B -->|master, release/*| C[Set version from pom.xml\nwithout -SNAPSHOT]
    B -->|develop, increment/*| D[Set version\nmajor.minor.qual.date_commitId_branch]
    C --> E[Build and deploy to Nexus]
    D --> E
    E --> F[Create git tag v-version]
    F --> G{Base branch?}
    G -->|increment/*, release/*| H[Create merge-pr/version tag]
    H --> I[Triggers merge-pr-tagged.yml]
    G -->|develop| J[Build changelog]
    J --> K[Create GitHub pre-release]
    G -->|other| L[Done]
```

### merge-pr-tagged.yml — Auto-merge Release PRs

**Trigger:** Push of a `merge-pr/*` tag

```mermaid
flowchart TD
    A[merge-pr/* tag pushed] --> B[Extract version from tag]
    B --> C{Version format?}
    C -->|major.minor.qualifier| D[Merge PR to master]
    D --> E[Triggers create-release-on-master.yml]
    C -->|other format| F[Squash PR to develop]
    F --> G[Triggers build.yml]
    D --> H[Delete merge-pr tag]
    F --> H
```

### create-release-on-master.yml — GitHub Release

**Trigger:** Push to `master` branch

```mermaid
flowchart TD
    A[Push to master] --> B[Get version from tag]
    B --> C[Build changelog]
    C --> D[Create GitHub release\nmarked as latest]
```

### release.yml — Manual Release Trigger

**Trigger:** Manual workflow dispatch with a version parameter (`auto` or explicit `major.minor.qualifier`)

```mermaid
flowchart TD
    A[Manual trigger\nwith version input] --> B{Version = auto?}
    B -->|yes| C[Read version from pom.xml\nstrip -SNAPSHOT]
    B -->|no| D[Use given version]
    C --> E[Set next version = qualifier + 1]
    D --> E
    E --> F[Create PR to master\nwith release version]
    E --> G[Create PR to develop\nwith next version]
    F --> H[Triggers build.yml]
    G --> H
```

### Workflow Interaction Overview

```mermaid
flowchart LR
    build[build.yml] -->|tag: merge-pr/*| merge[merge-pr-tagged.yml]
    merge -->|merge to master| release_master[create-release-on-master.yml]
    merge -->|squash to develop| build
    manual[release.yml\nmanual trigger] -->|creates PRs| build

    style manual fill:#ffd,stroke:#333
    style build fill:#ddf,stroke:#333
    style merge fill:#dfd,stroke:#333
    style release_master fill:#fdd,stroke:#333
```

### Other Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `build-dependabot.yml` | Dependabot PRs | Build and validate dependency updates |
| `bump-version.yml` | Manual dispatch | Increment project version |
| `create-release-tagged.yml` | Tag push `merge-pr/v*` | Create GitHub release on master for tagged versions |
| `jira-description-to-pr.yml` | PR creation | Sync JIRA ticket descriptions into PR body |
| `delete-old-draft-releases.yml` | Scheduled | Clean up stale draft releases |
| `sync-labels.yml` | Scheduled | Synchronize GitHub labels from configuration |

## How to Develop

Issue tracking uses [JIRA](https://blackbelt.atlassian.net/jira/dashboards).

> **Important:** There is no commit without a ticket number. Every pull request and commit message must include `JNG-XXX`.
