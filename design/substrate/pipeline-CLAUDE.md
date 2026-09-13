# pipeline/CLAUDE.md

## What This Does

Generates all artifacts needed to provision each platform: bootstrapper scripts, Dockerfiles, scaffold files, Kubernetes manifests, and ADL objects. Generate infrastructure only — never platform code.

## Pipeline Container

The agent runs inside a container built from `Dockerfile` at the repo root (the bootstrap pipeline file), launched via `run-agent.sh`. The repo is mounted at `/repo` and `work/` at `/work` — nothing is copied into the container.

## Read First

Read `platforms.yaml` at the repository root before generating anything. It has four sections: `def.families`, `def.runtimes`, `def.platforms.services`, and `def.platforms.runners`.

`def.families` declares the family base images:

```yaml
families:
  - name: jvm
    baseImage: eclipse-temurin:21
    description: JVM-based runtimes
  - name: node
    baseImage: node:lts-alpine
    description: Node.js-based runtimes
  - name: dotnet
    baseImage: mcr.microsoft.com/dotnet/sdk:8.0
    description: .NET-based runtimes
```

`def.platforms.services` declares service platforms:

```yaml
- name: typescript-react          # platform identifier used in all file names
  runtime: node
  family: node                    # jvm | node | dotnet — absent if no family image
  imageStyle: image               # image = official Docker Hub image exists
                                  # command = must download and install runtime
  installLib: github              # github | uri | source — only when imageStyle: command
  priority: 1
  domain:
    progLang: typescript          # always present
    platform: react               # absent for runner platforms
  engine: container
  resources:
    memoryMiB: 512
    cpuMillis: 500
  scaffold:
    files:
      - package.json
      - tsconfig.json
      - vite.config.ts
      - index.html
    editableSlot: src/
  service:
    port: 3000
    healthPath: /
  commands:
    - type: serve
      run: npm run dev
    - type: test
      run: npm test -- --watchAll=false
    - type: build
      run: npm run build
```

## Naming

Use `name` field as the platform identifier in all file and object names:
- `work/pipeline/typescript-react/Dockerfile`
- `work/substrates/typescript-react.yaml`
- `substrate-typescript-react-deps-v1`

## Generation Order

1. `work/{name}/`
2. `work/pipeline/{name}/Dockerfile`
3. `work/pipeline/{name}/scaffold/`
4. `work/pipeline/{name}/infra/pvc.yaml`
5. `work/substrates/{name}.yaml`
6. `work/substrates/{name}-crdtemplate.yaml`
7. `work/substrates/{name}-onboarding.yaml`
8. `work/pipeline/{name}/README.md`
9. `work/pipeline/images/{family}/Dockerfile`

## work/{name}/ (bootstrapper)

Generate for every platform in `def.platforms.services`: `entrypoint.sh`, `health.sh`, `README.md`.
Generate for every platform in `def.platforms.runners`: `run.sh`, `README.md`.

### work/{name}/entrypoint.sh

Service platforms only. Reads `RUNNERD_COMMAND` and maps it to the platform command. Replace `{commands[serve].run}`, `{commands[test].run}`, `{commands[build].run}` with the values from `commands` in `platforms.yaml`.

```bash
#!/bin/bash
. "/repo/assets/bootstrappers/lib/bootstrap.sh"
bootstrap_cd
bootstrap_forward_signals
case "${RUNNERD_COMMAND:-}" in
  "") echo "entrypoint.sh: RUNNERD_COMMAND is not set" >&2; exit 1 ;;
  serve) set -- {commands[serve].run} ;;
  test)  set -- {commands[test].run} ;;
  build) set -- {commands[build].run} ;;
  *) echo "unknown command: $RUNNERD_COMMAND" >&2; exit 1 ;;
esac
bootstrap_append_args "$@"
set -- "${_BOOTSTRAP_ARGS[@]}"
bootstrap_run_with_stdin "$@"
bootstrap_wait
```

Example for `typescript-react`:

```bash
#!/bin/bash
. "/repo/assets/bootstrappers/lib/bootstrap.sh"
bootstrap_cd
bootstrap_forward_signals
case "${RUNNERD_COMMAND:-}" in
  "") echo "entrypoint.sh: RUNNERD_COMMAND is not set" >&2; exit 1 ;;
  serve)
    if npm run 2>/dev/null | grep -q '^  dev$'; then
      set -- npm run dev
    else
      set -- npm start
    fi
    ;;
  start) set -- npm start ;;
  test)  set -- npm test ;;
  build) set -- npm run build ;;
  *)     set -- npm run "$RUNNERD_COMMAND" ;;
esac
bootstrap_append_args "$@"
set -- "${_BOOTSTRAP_ARGS[@]}"
bootstrap_run_with_stdin "$@"
bootstrap_wait
```

### work/{name}/health.sh

Service platforms only. Checks whether the service port is accepting connections.

```bash
#!/bin/bash
. "/repo/assets/bootstrappers/lib/bootstrap.sh"
bootstrap_health_check
```

### work/{name}/run.sh

Runner platforms only. Accepts a file path as `$1` and runs it using the platform invocation. Replace `{io.invocation}` with the value of `io.invocation` from `platforms.yaml`.

```bash
#!/bin/bash
. "/repo/assets/bootstrappers/lib/bootstrap.sh"
FILE="$1"
[ -z "$FILE" ] && { echo "run.sh: usage: run.sh <file>" >&2; exit 1; }
bootstrap_cd
bootstrap_forward_signals
set -- {io.invocation} "$FILE"
bootstrap_append_args "$@"
set -- "${_BOOTSTRAP_ARGS[@]}"
bootstrap_run_with_stdin "$@"
bootstrap_wait
```

Example for `javascript`/`typescript` — when the invocation depends on the file extension or available tools, use a `case` block instead of a single command:

```bash
#!/bin/bash
. "/repo/assets/bootstrappers/lib/bootstrap.sh"
FILE="$1"
[ -z "$FILE" ] && { echo "run.sh: usage: run.sh <file>" >&2; exit 1; }
bootstrap_cd
bootstrap_forward_signals
case "$FILE" in
  *.ts)
    if [ -x ./node_modules/.bin/tsx ]; then
      set -- ./node_modules/.bin/tsx "$FILE"
    elif [ -x ./node_modules/.bin/ts-node ]; then
      set -- ./node_modules/.bin/ts-node "$FILE"
    elif command -v ts-node >/dev/null 2>&1; then
      set -- ts-node "$FILE"
    else
      set -- node --experimental-strip-types "$FILE"
    fi
    ;;
  *)
    set -- node "$FILE"
    ;;
esac
bootstrap_append_args "$@"
set -- "${_BOOTSTRAP_ARGS[@]}"
bootstrap_run_with_stdin "$@"
bootstrap_wait
```

### Constraints

- Always `#!/bin/bash`
- Always source `/repo/assets/bootstrappers/lib/bootstrap.sh`
- Always `bootstrap_forward_signals` + `bootstrap_wait`
- Never `exec` — bootstrapper must stay alive as PID 1
- No buffering on stdout/stderr

### Library Functions

| Function | Purpose |
|---|---|
| `bootstrap_cd` | Change to `RUNNERD_WORK_DIR` (default `/app/src`) |
| `bootstrap_append_args` | Set `_BOOTSTRAP_ARGS` array to `"$@"` + `RUNNERD_ARGS` |
| `bootstrap_forward_signals` | Install SIGTERM/SIGINT traps forwarding to child process group |
| `bootstrap_run_with_stdin CMD [ARGS...]` | Launch CMD, pipe `RUNNERD_STDIN` if set, set `_BOOTSTRAP_CHILD_PID` |
| `bootstrap_wait` | Wait for `_BOOTSTRAP_CHILD_PID`, exit with its status |
| `bootstrap_health_check` | Check `RUNNERD_HEALTH_PORT`, bounded to 1.8s |

## work/pipeline/{name}/Dockerfile

One Dockerfile per platform. Which template to use depends on the platform's `imageStyle` and `family` fields.

### Service platform — `imageStyle: image`

Applies to entries in `def.platforms.services` where `imageStyle: image`. Replace `{official image}` with the appropriate image tag, and `{dependency install command}` with the command that pre-installs dependencies from the scaffold files (e.g. `npm install`, `mvn dependency:resolve -q`).

```dockerfile
FROM {official image}
WORKDIR /app
COPY work/pipeline/{name}/scaffold/ /app/
RUN {dependency install command}
COPY work/{name}/entrypoint.sh /bootstrapper/entrypoint.sh
COPY work/{name}/health.sh /bootstrapper/health.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
ENTRYPOINT ["/bootstrapper/entrypoint.sh"]
```

Example for `typescript-react`:

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY work/pipeline/typescript-react/scaffold/ /app/
RUN npm install
COPY work/typescript-react/entrypoint.sh /bootstrapper/entrypoint.sh
COPY work/typescript-react/health.sh /bootstrapper/health.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
ENTRYPOINT ["/bootstrapper/entrypoint.sh"]
```

### Runner platform — `imageStyle: image`

Applies to entries in `def.platforms.runners` where `imageStyle: image`.

```dockerfile
FROM {official image}
COPY work/{name}/run.sh /bootstrapper/run.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
```

Example for `python`:

```dockerfile
FROM python:3-alpine
COPY work/python/run.sh /bootstrapper/run.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
```

### `imageStyle: command` without `family`

Applies to entries in `def.platforms.services` or `def.platforms.runners` where `imageStyle: command` and no `family` is set. The runtime must be downloaded using the install library indicated by `installLib`.

Two libraries are available:

**`github.sh`** — downloads a release asset from a GitHub repository. Use when the runtime is distributed as a GitHub release.
```sh
. /lib/github.sh
github_download {owner}/{repo} {asset-filename} [version]
```

**`uri.sh`** — downloads an asset from a direct URL. Use when the runtime is distributed via a standalone download URL.
```sh
. /lib/uri.sh
uri_download {url} [output-filename]
```

```dockerfile
FROM debian:stable-slim
COPY assets/pipeline/lib/ /lib/
# Check for latest: {upstream release URL}
ENV {RUNTIME}_VERSION={latest stable version}
RUN . /lib/{installLib}.sh && \
    {download and install steps}
COPY work/{name}/entrypoint.sh /bootstrapper/entrypoint.sh
COPY work/{name}/health.sh /bootstrapper/health.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
ENTRYPOINT ["/bootstrapper/entrypoint.sh"]
```

### `imageStyle: command` with `family`

Applies to entries in `def.platforms.services` or `def.platforms.runners` where `imageStyle: command` and a `family` is set. Do not generate a per-platform Dockerfile. The runtime is installed in the shared family base image at `work/pipeline/images/{family}/Dockerfile`.

## work/pipeline/{name}/scaffold/

Only for `def.platforms.services`. Generate one file per entry in `scaffold.files`. Infrastructure files the platform owns — baked into the container image and placed in the scaffold PVC for Theia environments.

`scaffold.editableSlot` is NOT scaffold — it is where EiBot generates learner code. Do not generate it here.

Generate realistic, working content. The platform must start correctly when the container image is built.

## work/pipeline/{name}/infra/pvc.yaml

Only for `def.platforms.services`. Three PVCs, all `ReadOnlyMany`. PVC names derived from `name` field:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: substrate-{name}-scaffold-v1
spec:
  accessModes: [ReadOnlyMany]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: substrate-{name}-deps-v1
spec:
  accessModes: [ReadOnlyMany]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: substrate-{name}-plugins-v1
spec:
  accessModes: [ReadOnlyMany]
  resources:
    requests:
      storage: 1Gi
```

Version suffix `v1` must be consistent across `pvc.yaml`, `substrate-onboarding`, and `crdtemplate`.

## work/substrates/{name}.yaml

What runnerd reads at session time. All values derived from `platforms.yaml`.

- `sourceDir` — `/app/` + `scaffold.editableSlot`
- `filesystem.structure` — always two entries: `/app/deps` (readOnly) and `/app/scaffold` (readOnly)
- `env` — environment variables appropriate for the runtime and framework (e.g. port, mode). Derive from the platform's known conventions.
- `endpoints` — only for service platforms (`def.platforms.services`). Omit for runners.
- `commands` — copied from the platform's `commands` entries.

```yaml
model: substrate
metadata:
  name: {name}
def:
  domain:
    progLang: {domain.progLang}
    platform: {domain.platform}
  engine: container
  image: reg.jobico.local/runnerd-{name}:latest
  resources:
    memoryMiB: {resources.memoryMiB}
    cpuMillis: {resources.cpuMillis}
  env:
    {KEY}: "{value}"
  sourceDir: /app/{scaffold.editableSlot}
  filesystem:
    structure:
      - path: /app/deps
        readOnly: true
      - path: /app/scaffold
        readOnly: true
  endpoints:
    - port: {service.port}
      healthPath: {service.healthPath}
  commands:
    - type: serve
      run: {commands[serve].run}
    - type: test
      run: {commands[test].run}
    - type: build
      run: {commands[build].run}
```

Example for `kotlin-ktor`:

```yaml
model: substrate
metadata:
  name: kotlin-ktor
def:
  domain:
    progLang: kotlin
    platform: ktor
  engine: container
  image: reg.jobico.local/runnerd-kotlin-ktor:latest
  resources:
    memoryMiB: 512
    cpuMillis: 500
  env:
    PORT: "8080"
  sourceDir: /app/src/main/kotlin
  filesystem:
    structure:
      - path: /app/deps
        readOnly: true
      - path: /app/scaffold
        readOnly: true
  endpoints:
    - port: 8080
      healthPath: /
  commands:
    - type: serve
      run: ./gradlew run
    - type: test
      run: ./gradlew test
    - type: build
      run: ./gradlew build
```

## work/substrates/{name}-crdtemplate.yaml

Only for `def.platforms.services`. Complete Kubernetes environment CRD for Challenge (Theia) environments. The generated file must be complete and ready to use.

**Hardcoded** (derived from `platforms.yaml` and `substrate-onboarding`):
- Container port — `service.port`
- PVC names — copy exactly from `substrate-onboarding.pvcs.*.name`
- `initJob.commands` — `ln -s /mnt/deps/{item} /workspace/project/{item}` for each dep the platform needs in the workspace
- Resource limits — from `resources.memoryMiB` and `resources.cpuMillis`

**Runtime variables** (resolved by the environment operator — do not change):
`{&environment-name}`, `{&user-id}`, `{&infra.namespace}`, `{&infra.redis.host}`, `{&infra.redis.port}`, `{&infra.cloud.uri}`, `{&runner-account-name}`, `{&cleaner-account-name}`, `{&cleaner-image}`, `{&environment-ttl-schedulle}`, `{&ttl-timezone}`, `{&ttl-string}`, `{&capability-id}`, `{&environment-type}`, `{&ENV_JWKS_URI}`, `{&ENV_JWT_ISSUER}`, `{&ENV_JWT_AUDIENCE}`

```yaml
model: crdtemplate
metadata:
  name: {name}
  extra:
    archetype: environment
def:
  domain:
    progLang: {domain.progLang}
    platform: {domain.platform}
  content:
    group: sandbox.eivo.ca
    version: v1
    kind: Environment
    plural: environments
    metadata:
      name: {&environment-name}
    spec:
      cleaner:
        serviceAccountName: {&cleaner-account-name}
        schedule: "{&environment-ttl-schedulle}"
        timeZone: "{&ttl-timezone}"
        expirationString: "{&ttl-string}"
        image: {&cleaner-image}
      project:
        name: {&environment-name}
        projectDirSize: "5Gi"
        projectDir: /workspace/project
      initJob:
        commands:
          - ln -s /mnt/deps/{item} /workspace/project/{item}
          - chown -R 1000:1000 /workspace/project
        supportMount:
          mount:
            name: depsdir
            mountPath: /mnt/deps
          volume:
            name: depsdir
            persistentVolumeClaim:
              claimName: {substrate-onboarding.pvcs.deps.name}
        spec:
          spec:
            template:
              spec:
                imagePullSecrets:
                  - name: reg-cred-secret
                containers:
                  - name: init-container
                    image: reg.jobico.local/lingv-environment-job:latest
                    imagePullPolicy: Always
                    serviceAccountName: {&runner-account-name}
                    env:
                      - name: USER_ID
                        value: "{&user-id}"
                      - name: NAMESPACE
                        value: "{&infra.namespace}"
                      - name: KV_STORE_HOST
                        value: "{&infra.redis.host}"
                      - name: KV_STORE_PORT
                        value: "{&infra.redis.port}"
                      - name: EIVO_API
                        value: "{&infra.cloud.uri}"
                      - name: CAPABILITY_ID
                        value: "{&capability-id}"
                      - name: ENVIRONMENT_TYPE
                        value: "{&environment-type}"
                restartPolicy: OnFailure
            backoffLimit: 3
      app:
        image: reg.jobico.local/lingv-environment-ide:latest
        ingressPort: 80
        command: ["npm", "run", "start:browser"]
        imagePullSecrets: ["reg-cred-secret"]
        ports:
          - port:
              containerPort: {service.port}
              protocol: TCP
              name: http
        serviceAccountName: {&runner-account-name}
        podSecurityContext:
          runAsUser: 1000
          runAsGroup: 1000
          fsGroup: 1000
        containerSecurityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
        resources:
          requests:
            memory: "{resources.memoryMiB}Mi"
            cpu: "{resources.cpuMillis}m"
          limits:
            memory: "{resources.memoryMiB * 2}Mi"
            cpu: "1500m"
        env:
          - name: HOME
            value: /home/theia
          - name: JWKS_URI
            value: "{&ENV_JWKS_URI}"
          - name: JWT_ISSUER
            value: "{&ENV_JWT_ISSUER}"
          - name: JWT_AUDIENCE
            value: "{&ENV_JWT_AUDIENCE}"
          - name: USE_MOCK
            value: "false"
          - name: KV_STORE_HOST
            value: "{&infra.redis.host}"
          - name: KV_STORE_PORT
            value: "{&infra.redis.port}"
          - name: EIVO_API
            value: "{&infra.cloud.uri}"
        extraVolumes:
          - mount:
              name: scaffold
              mountPath: /mnt/scaffold
              readOnly: true
            volume:
              name: scaffold
              persistentVolumeClaim:
                claimName: {substrate-onboarding.pvcs.scaffold.name}
          - mount:
              name: plugins
              mountPath: /home/theia/.theia/plugins
              readOnly: true
            volume:
              name: plugins
              persistentVolumeClaim:
                claimName: {substrate-onboarding.pvcs.plugins.name}
          - mount:
              name: tmp-volume
              mountPath: /tmp
            volume:
              name: tmp-volume
              emptyDir:
                medium: Memory
          - mount:
              name: theia-config-volume
              mountPath: /home/theia/.theia
            volume:
              name: theia-config-volume
              emptyDir:
                medium: Memory
```

## work/substrates/{name}-onboarding.yaml

Onboarding agent entry point. PVC names here are the single source of truth — `crdtemplate` must copy them exactly.

Two distinct sets of files come from `scaffold.files`:

**All scaffold files** → `pvcs.scaffold` — the full set of files in `work/pipeline/{name}/scaffold/`, copied as-is into the scaffold PVC.

**Dependency manifest files** → `pvcs.deps.def.files` — the subset of `scaffold.files` that declare dependencies for the platform's package manager. The package manager runs against these files inside the deps PVC to pre-install dependencies. Select only:

| Package manager | Files |
|---|---|
| npm | `package.json`, `package-lock.json` |
| Maven | `pom.xml` |
| Gradle | `build.gradle.kts`, `settings.gradle.kts`, `gradle.properties` |
| Cargo | `Cargo.toml`, `Cargo.lock` |
| Composer | `composer.json`, `composer.lock` |
| Bundler | `Gemfile`, `Gemfile.lock` |
| pip | `requirements.txt` |
| sbt | `build.sbt`, `project/plugins.sbt` |
| mix | `mix.exs`, `mix.lock` |

Only include files that are present in `scaffold.files` for this platform.

```yaml
model: substrate-onboarding
metadata:
  name: {name}
def:
  substrate: {name}
  dockerfile: work/pipeline/{name}/Dockerfile
  pvcs:
    deps:
      name: substrate-{name}-deps-v1
      type: install
      def:
        files:
          - source: work/pipeline/{name}/scaffold/{manifest file}
            dest: /mnt/deps/{manifest file}
        commands:
          - {dependency install command — e.g. npm install, mvn dependency:resolve}
    scaffold:
      name: substrate-{name}-scaffold-v1
      path: work/pipeline/{name}/scaffold/
    plugins:
      name: substrate-{name}-plugins-v1
      plugins:
        - id: {publisher}.{extension-name}
          source: open-vsx
  manifests:
    - work/pipeline/{name}/infra/pvc.yaml
```

## work/pipeline/images/{family}/Dockerfile

Generated once per entry in `def.families`. Builds the family base image that `imageStyle: command` platforms extend.

**Source:** Read `def.families` for the base image. Read all `def.platforms.services` and `def.platforms.runners` entries where `imageStyle: command` and `family` matches — these are the runtimes to install.

**Structure:**

```dockerfile
# BEGIN FIXED ---------------------------------------------------------------
FROM {def.families[name].baseImage}
COPY assets/pipeline/lib/ /lib/
# END FIXED -----------------------------------------------------------------

# BEGIN RUNTIMES ------------------------------------------------------------
# One labeled block per platform entry with imageStyle: command and this family.
# Use assets/pipeline/lib/{installLib}.sh for each runtime.

# [{name}]
# Check for latest: {upstream release URL}
ENV {RUNTIME}_VERSION={latest version}
RUN . /lib/{installLib}.sh && \
    {download and install steps}

# END RUNTIMES --------------------------------------------------------------
```

**Rules:**

- One `# [{name}]` block per matching platform — both service and runner platforms
- Multiple platforms sharing the same `runtime` (e.g. `kotlin-ktor` and `kotlin`) get one block between them — deduplicate by `runtime`
- `COPY assets/pipeline/lib/` is in FIXED — always present, never repeated per runtime
- No bootstrapper scripts, no scaffold, no entrypoint — this image is a base only

## work/pipeline/{name}/README.md

Documents: platform identity, capabilities, scaffold files and their purpose, editable slot, how to build the container image manually.