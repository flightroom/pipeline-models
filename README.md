# Flightroom pipeline models

The CI pipelines that Flightroom copies into every frontend and backend repository it creates, one per
kind of application, language and CI provider. They are written for enterprise setups: Azure DevOps with
only built-in tasks (no Marketplace extension), self-hosted agents and runners, private registries, and
proxies (no `curl | bash` installers).

## Layout

```
<type>/<language>/<provider>/
```

| type      | what it builds                                                    | languages                    |
|-----------|-------------------------------------------------------------------|------------------------------|
| `service` | a backend, shipped as a container image                           | `java`, `python`, `javascript` |
| `host`    | a frontend served by its own container (nginx serving its build)  | `javascript`                 |
| `remote`  | a frontend served by Flightroom as a static bundle                 | `javascript`                 |

| provider        | file in this repository        | becomes, in the new repository  |
|-----------------|--------------------------------|---------------------------------|
| `azure_dev_ops` | `azure-pipelines.yml`          | `azure-pipelines.yml`           |
| `gitlab`        | `.gitlab-ci.yml`               | `.gitlab-ci.yml`                |
| `github`        | `workflows/build-and-release.yml` | `.github/workflows/build-and-release.yml` (the template's own workflows are removed) |

```
service/java/{azure_dev_ops,gitlab,github}        Spring Boot, Quarkus, Micronaut; Maven or Gradle,
                                                  picked from the wrapper present (mvnw, gradlew)
service/python/{azure_dev_ops,gitlab,github}      FastAPI and the like; pip, with constraints.txt,
                                                  requirements*.txt and the "dev" extra when present
service/javascript/{azure_dev_ops,gitlab,github}  Node.js / TypeScript; pnpm in the version of packageManager
host/javascript/{azure_dev_ops,gitlab,github}     Vite, webpack, single-spa hosts with a Dockerfile
remote/javascript/{azure_dev_ops,gitlab,github}   Vite, webpack, Angular remotes; output detected
                                                  (dist, build, out) or set by hand
```

One pipeline per language serves every template of that language. Flightroom looks a template up as
`<type>/<compiler>/<template slug>/<provider>`, then `<type>/<compiler>/<provider>`, then
`<type>/<language>/<provider>`: to give one template its own pipeline, add a folder at one of the first
two places. A template with no folder here (a Go or .NET backend, say) keeps the pipeline it ships with.

## What every pipeline does

- **Push to the main branch, pull or merge request:** install, build and test. Test results in JUnit
  format are published where the build writes them (Maven Surefire/Failsafe, Gradle, pytest; for
  JavaScript, a test script writing `test-results/**/*.xml` or `junit*.xml`).
- **Tag `<version>`:** the same, then the release. The tag must be a valid version: letters, digits,
  `.`, `_`, `-`, at most 128, not starting with `.` or `-` (e.g. `1.4.0`, `v2.0.0-rc.1`).
  - `service` and `host`: `docker build` from the repository's Dockerfile, push
    `<registry>/<prefix>/<slug>:<version>`, then register the version in Flightroom:

    ```
    POST <domain>/api/backends/by-slug/<slug>/versions/<version>     (service)
    POST <domain>/api/frontends/by-slug/<slug>/versions/<version>    (host)
    api-key: $FLIGHTROOM_API_KEY
    Content-Type: application/json

    {"image": "<registry>/<prefix>/<slug>:<version>", "digest": "sha256:…", "commitSha": "<sha>"}
    ```

    `digest` and `commitSha` are sent when known. The API key must have the MANAGER role.
  - `remote`: zip the build output (its files at the root of the zip) and upload it:

    ```
    POST <domain>/api/frontends/by-slug/<slug>/upload/<version>
    api-key: $FLIGHTROOM_API_KEY
    Content-Type: multipart/form-data; field "file" = bundle.zip (application/zip)
    ```

    This is the same call the `flightroom-upload` Azure DevOps task makes, done with curl.

`<domain>` is the Flightroom URL. A trailing `/` or `/api` is dropped before `/api/...` is added.

Values that come from the CI (the tag, variables, secrets) reach the scripts through environment variables
only. They are never pasted into a script body, where a crafted tag could inject shell. The API key goes to
curl through stdin, so it never shows in the process list of a shared agent.

### Placeholders

Flightroom replaces these when it copies a pipeline into a new repository:

| placeholder           | becomes                                                       |
|-----------------------|---------------------------------------------------------------|
| `%serviceSlug%`       | the slug of the backend                                       |
| `%microfrontendSlug%` | the slug of the frontend                                      |
| `%domain%`            | the URL of the Flightroom console; a CI variable can override it |

## Point a Flightroom project at this repository

1. Push this repository to GitHub, GitLab or Azure DevOps, reachable through one of the code repository
   connections of the Flightroom project.
2. In Flightroom, open the project's **Settings → Pipeline templates** (owners only), turn on **Use the
   pipeline templates of a repository of the project**, pick the connection and this repository, and
   optionally a branch or tag (the default branch when left empty; a tag freezes the version used).
3. Repositories created from then on get their pipeline from here. Repositories that already exist keep
   theirs.

Change defaults (agent pool, tool versions, registry) here, once. Every repository created after the change
gets them.

## Set up once, per provider

### Azure DevOps

| what | name | notes |
|------|------|-------|
| Variable group | `MFE_ORCHESTRATOR_SECRETS` | Pipelines → Library. Authorize it for the pipelines, or for all of them. |
| ↳ secret variable | `FLIGHTROOM_API_KEY` | Flightroom API key of the project, MANAGER role |
| ↳ variable (optional) | `FLIGHTROOM_DOMAIN` | Flightroom URL, if not the console that created the repository |
| ↳ variable (optional) | `FLIGHTROOM_REGISTRY_HOST` | registry host in the image reference sent to Flightroom, e.g. `acme.azurecr.io`. Left out, it is read from the image Docker@2 pushed. |
| ↳ variable (optional) | `FLIGHTROOM_IMAGE_REPOSITORY_PREFIX` | path before the slug: `platform` gives `platform/<slug>` |
| Service connection | `flightroom-registry` | Docker Registry type (ACR, Harbor, Artifactory, Nexus, Docker Hub…). Used by service and host. |

Parameters at the top of each `azure-pipelines.yml`, compiled into the pipeline (change their defaults here):

| parameter | default | |
|-----------|---------|-|
| `agentPool` | `Azure Pipelines` | the Microsoft-hosted pool; put a self-hosted pool's name here |
| `vmImage` | `ubuntu-latest` | Microsoft-hosted only, ignored on a self-hosted pool |
| `registryServiceConnection` | `flightroom-registry` | a parameter, not a variable: Azure DevOps authorizes service connections when it compiles the pipeline |
| `javaVersion` / `pythonVersion` / `nodeVersion` | `25` / `3.15` / `24.x` | |
| `buildOutput` | `dist` (host), `auto` (remote) | the folder the Dockerfile copies, or the folder to upload |

Self-hosted agents: Linux with bash, curl and Docker (service and host). Java needs the JDK pre-installed
(`JAVA_HOME_25_X64`), Python needs the version in the agent's tool cache. Behind a proxy, configure the agent's
proxy, point the Maven/Gradle wrapper `distributionUrl` to an internal mirror, and set `PIP_INDEX_URL`,
an `.npmrc` registry and `COREPACK_NPM_REGISTRY` as needed.

Azure Repos ignores the `pr:` block: add a build validation policy on `main` to build pull requests.

### GitHub Actions

Set on the organization (Settings → Secrets and variables → Actions) or on each repository:

| kind | name | default |
|------|------|---------|
| secret | `FLIGHTROOM_API_KEY` | required; Flightroom API key of the project, MANAGER role |
| variable | `FLIGHTROOM_DOMAIN` | the console that created the repository |
| variable | `FLIGHTROOM_REGISTRY` | `ghcr.io` |
| variable | `FLIGHTROOM_IMAGE_REPOSITORY_PREFIX` | the repository owner |
| variable | `FLIGHTROOM_REGISTRY_USERNAME` | the workflow's actor |
| secret | `FLIGHTROOM_REGISTRY_PASSWORD` | `GITHUB_TOKEN`, enough for ghcr.io |
| variable | `FLIGHTROOM_RUNNER` | `ubuntu-latest`; a self-hosted runner label otherwise |
| variable | `FLIGHTROOM_JAVA_VERSION` / `FLIGHTROOM_PYTHON_VERSION` / `FLIGHTROOM_NODE_VERSION` | `25` / `3.15` / `24` |
| variable | `FLIGHTROOM_BUILD_OUTPUT` | host: `dist`; remote: the first of `dist`, `build`, `out` |

Workflow permissions are `contents: read`, plus `packages: write` on the release job of service and host.
Actions are pinned to their major version.

### GitLab CI

Set as CI/CD variables of the group (Settings → CI/CD → Variables), or of each project:

| variable | default |
|----------|---------|
| `FLIGHTROOM_API_KEY` | required, masked; Flightroom API key of the project (MANAGER role for service and host) |
| `FLIGHTROOM_DOMAIN` | the console that created the repository |
| `FLIGHTROOM_IMAGE_BUILDER` | `dind` (a `docker:dind` service, needs a privileged runner), `docker` (the runner's own daemon: mounted socket or shell runner) or `kaniko` (no daemon, no privileged mode) |
| `FLIGHTROOM_IMAGE_REPOSITORY` | full repository without tag; empty: `<FLIGHTROOM_IMAGE_PREFIX>/<slug>`, else the project's own registry `$CI_REGISTRY_IMAGE` |
| `FLIGHTROOM_IMAGE_PREFIX` | empty; e.g. `registry.acme.com/platform` (include the registry host: the login goes to the host of the repository) |
| `FLIGHTROOM_REGISTRY_USER` / `FLIGHTROOM_REGISTRY_PASSWORD` | GitLab's job credentials `$CI_REGISTRY_USER` / `$CI_REGISTRY_PASSWORD` |
| `FLIGHTROOM_BUILD_OUTPUT` | host: `dist`; remote: the first of `dist`, `build`, `out` |
| `FLIGHTROOM_BUILD_IMAGE` | `eclipse-temurin:25-jdk`, `python:3.15` or `node:24` |
| `FLIGHTROOM_DOCKER_IMAGE` / `FLIGHTROOM_DIND_IMAGE` / `FLIGHTROOM_KANIKO_IMAGE` / `FLIGHTROOM_CURL_IMAGE` | `docker:29`, `docker:29-dind`, `gcr.io/kaniko-project/executor:v1.23.2-debug`, `curlimages/curl:8.11.1`: point them to a mirror if runners cannot reach Docker Hub |

Group and project variables win over the defaults written in `.gitlab-ci.yml`. Pipelines run on merge
requests, on the default branch and on tags.

Kaniko is archived upstream, and it treats BuildKit features such as `RUN --mount=type=cache` (used by the
Java templates' Dockerfiles) only partially. Prefer `dind` or `docker` where the runners allow it.
