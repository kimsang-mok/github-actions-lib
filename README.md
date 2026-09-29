# Miracode GitHub Actions Library

A reusable, **build-tool agnostic** CI + release pipeline.

Instead of a single script full of `if [ tool = maven ] … elif …`, each supported
build tool is implemented as its own set of **provider composite actions** that
share one uniform contract. The pipeline resolves the tool once and then routes
each stage to the matching provider through declarative `if:` conditions.

## Supported build tools

| Tool   | Detected by (in order)                                        | Runtime installed | Version read from                                           |
| ------ | ------------------------------------------------------------- | ----------------- | ----------------------------------------------------------- |
| maven  | `pom.xml`                                                     | JDK (temurin)     | `project.version` via `help:evaluate`                       |
| gradle | `build.gradle`, `build.gradle.kts`, `settings.gradle(.kts)`   | JDK (temurin)     | Gradle`project.version` (init script)                       |
| npm    | `package.json`                                                | Node.js           | `package.json` `version`                                    |
| python | `pyproject.toml`, `setup.py`, `setup.cfg`, `requirements.txt` | Python            | `pyproject.toml` / `setup.cfg` / `setup.py` / `__init__.py` |

Detection can be overridden with the `build-tool` input (`maven|gradle|npm|python`).

## Architecture

```
.github/actions/
  detect/                      # resolve build-tool (explicit or auto-detect)
  setup/{maven,gradle,npm,python}/        # install runtime
  version/{maven,gradle,npm,python}/      # resolve group/artifact/version + image tag
  build/{maven,gradle,npm,python}/        # build + test + upload reports
  version-bump/{maven,gradle,npm,python}/ # write a version into the manifest
  initialization/              # orchestrator: checkout + routed setup + routed version
  docker-build/                # checkout + routed build + docker build/push
  docker-promote/              # re-tag & push an existing image
  release/                     # git-flow release finish (routes version-bump)
  post-process/                # always-run summary / notification
```

Every provider implements the same **uniform contract**, so the workflow is
tool-agnostic:

- `setup/<tool>` — inputs `runtime-version`, `working-directory`
- `version/<tool>` — outputs `group-id`, `artifact-id`, `version`, `image-tag`
- `build/<tool>` — inputs `working-directory`, `skip-tests`
- `version-bump/<tool>` — inputs `version`, `working-directory`

The only place branching is unavoidable is `detect/` — it decides _which_
provider to use. The providers themselves contain no build-tool branching.

## Usage

```yaml
name: Release
on:
  push:
    branches: [main]

jobs:
  ci:
    uses: kimsang-mok/github-actions-lib/.github/workflows/ci-release.yml@main
    with:
      image-name: my-org/my-app
      # build-tool: auto      # or maven | gradle | npm | python
      # runtime-version: '21' # JDK / Node / Python version (per-tool default otherwise)
      # working-directory: .
      # release-branch: main
      # develop-branch: develop
    secrets: inherit # needs REGISTRY_USERNAME, REGISTRY_PASSWORD (+ RELEASE_GIT_TOKEN)
```

Add a `Dockerfile` (path configurable via the `dockerfile` input) in
`working-directory` that packages the built artifact.

### Inputs

| Input               | Default      | Description                                               |
| ------------------- | ------------ | --------------------------------------------------------- |
| `build-tool`        | `auto`       | `maven` \| `gradle` \| `npm` \| `python` \| `auto`        |
| `runtime-version`   | `""`         | JDK / Node / Python version (empty = provider default)    |
| `java-version`      | `""`         | Deprecated alias, used as a fallback for`runtime-version` |
| `working-directory` | `.`          | Folder containing the build manifest                      |
| `image-name`        | (required)   | Image name without registry/tag                           |
| `registry`          | `ghcr.io`    | Container registry host                                   |
| `dockerfile`        | `Dockerfile` | Dockerfile path, relative to`working-directory`           |
| `release-branch`    | `main`       | Receives the released (non-SNAPSHOT) version              |
| `develop-branch`    | `develop`    | Receives the next development snapshot                    |
| `skip-tests`        | `true`       | Skip unit tests during the build stage                    |
| `skip-release`      | `false`      | Skip the release (close version) stage                    |
| `skip-promote`      | `false`      | Skip the docker promote stage                             |

### Secrets

| Secret              | Required | Description                                      |
| ------------------- | -------- | ------------------------------------------------ |
| `REGISTRY_USERNAME` | yes      | Container registry username                      |
| `REGISTRY_PASSWORD` | yes      | Container registry password/token                |
| `RELEASE_GIT_TOKEN` | no       | PAT used to push tags/merges (else GITHUB_TOKEN) |
| `NOTIFY_WEBHOOK`    | no       | Webhook called by the post stage                 |

### Outputs

| Output             | Description                      |
| ------------------ | -------------------------------- |
| `build-tool`       | Resolved build tool              |
| `version`          | Project version                  |
| `image-ref`        | Pushed CI image reference        |
| `released-version` | Version released by the pipeline |

## Conventions

- **Version must be set** in the project manifest (Gradle: `version =` in
  `gradle.properties`/`build.gradle`; Python: a static `version`).
- **Image tag** is `<version>-<short-sha>`; the release stage promotes it to the
  bare released version plus `latest`.
- **Test reports** are uploaded from the provider-specific location
  (`target/surefire-reports`, `build/test-results`, `junit.xml`).
- The release stage assumes a classic **git-flow** layout: a `develop` branch
  and a `release/<version>` branch that is merged into `main` and back into
  `develop`.

## Notes / limitations

- Detection order is `maven > gradle > npm > python`; set `build-tool`
  explicitly if a repo contains more than one manifest.
- `-SNAPSHOT` release suffixes are valid npm prereleases; the released version
  is the current version with any `-SNAPSHOT` suffix stripped.
- The pipeline references composite actions with relative paths
  (`$/.github/actions/...`), which resolve against this repository.
