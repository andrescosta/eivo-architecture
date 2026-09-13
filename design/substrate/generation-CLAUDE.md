# generation/CLAUDE.md

## What This Does

Generates all assets needed to onboard each domain into Eivo. Read `domains.yaml` first — it is the single source of truth. Generate infrastructure only — never platform code.

## Generation Container

The agent runs inside a container built from `Dockerfile` at the repo root, launched via `run-agent.sh`. The repo is mounted at `/repo` and `work/` at `/work` — nothing is copied into the container.

## Read First

Read `domains.yaml` → `def.domains`. It is a flat list. Each entry has a `for` field that declares which functions the platform supports:

- `environment-domain-ide` — Theia-based environment (PVCs, plugins, crdtemplate)
- `service` — long-lived process on a port, lifecycle managed by runnerd
- `program` — one-time execution with output, run by runnerd

A platform can support multiple functions. Generate assets for every function listed in `for`.

```yaml
- name: typescript-react
  runtime: node
  family: node
  imageStyle: image
  priority: 0
  for:
    - environment-domain-ide
    - service
  domain:
    progLang: typescript
    platform: react
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

- name: java
  runtime: java
  family: jvm
  imageStyle: image
  priority: 0
  for:
    - environment-domain-ide
    - program
  domain:
    progLang: java
  resources:
    memoryMiB: 256
    cpuMillis: 250
  scaffold:
    files:
      - pom.xml
    editableSlot: src/main/java/
  io:
    extension: .java
    invocation: javac + java
```

## Naming

Use `name` as the platform identifier in all file and directory names:
- `work/runs/{run-name}/generation/typescript-react/Dockerfile`
- `work/runs/{run-name}/substrates/typescript-react.yaml`
- `substrate-typescript-react-deps-v1`

## Generation Order

| # | Output | `for` | Log key (`work/runs/{run-name}/log.yaml`) |
|---|---|---|---|
| 1 | `work/runs/{run-name}/{name}/` | `service` or `program` | `bootstrapper` |
| 2 | `work/runs/{run-name}/generation/{name}/Dockerfile` | all | `dockerfile` |
| 3 | `work/runs/{run-name}/generation/{name}/scaffold/` | `environment-domain-ide` | `scaffold` |
| 4 | `work/runs/{run-name}/generation/{name}/infra/pvc.yaml` | `environment-domain-ide` | `pvc` |
| 5 | `work/runs/{run-name}/substrates/{name}.yaml` | all | `substrate` |
| 6 | `work/runs/{run-name}/substrates/{name}-crdtemplate.yaml` | `environment-domain-ide` | `crdtemplate` |
| 7 | `work/runs/{run-name}/substrates/{name}-onboarding.yaml` | `environment-domain-ide` | `onboarding` |
| 8 | `work/runs/{run-name}/generation/{name}/README.md` | all | `readme` |
| 9 | `work/runs/{run-name}/generation/images/{family}/Dockerfile` | all (`imageStyle: command` only) | `families.{family}.dockerfile` |

Skip a step if the platform's `for` list does not include the required function.

---

## environment-domain-ide

Generate for every platform where `environment-domain-ide` is in `for`.

### work/runs/{run-name}/generation/{name}/scaffold/

Generate one file per entry in `scaffold.files`. These are the infrastructure files baked into the container image and placed in the scaffold PVC for Theia environments.

`scaffold.editableSlot` is NOT scaffold — it is where EiBot generates learner code. Do not generate it here.

Generate realistic, working content. The platform must start correctly when the container image is built.

### work/runs/{run-name}/generation/{name}/infra/pvc.yaml

Three PVCs, all `ReadOnlyMany`. PVC names derived from `name`:

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

### work/runs/{run-name}/substrates/{name}-crdtemplate.yaml

Complete Kubernetes environment CRD for Theia-based environments. The generated file must be complete and ready to use.

**Hardcoded** (derived from `domains.yaml` and `substrate-onboarding`):
- Container port — `service.port` (use 8080 as default if platform has no `service`)
- PVC names — copy exactly from `substrate-onboarding.pvcs.*.name`
- `initJob.commands` — read `packageManager` from the platform entry. If absent, no deps PVC exists — only generate `chown`. Otherwise use the table below to determine the symlink command:

| `packageManager` | symlink command |
|---|---|
| `npm` | `ln -s /mnt/deps/node_modules /workspace/project/node_modules` |
| `maven` | `ln -s /mnt/deps/.m2 /root/.m2` |
| `gradle` | `ln -s /mnt/deps/.gradle /root/.gradle` |
| `pip` | `ln -s /mnt/deps/site-packages /usr/local/lib/python3/dist-packages` |
| `cargo` | `ln -s /mnt/deps/registry /root/.cargo/registry` |
| `bundler` | `ln -s /mnt/deps/bundle /usr/local/bundle` |
| `sbt` | `ln -s /mnt/deps/.ivy2 /root/.ivy2` |
| `mix` | `ln -s /mnt/deps/_build /workspace/project/_build` |
| `nuget` | `ln -s /mnt/deps/nuget /root/.nuget` |
| `composer` | `ln -s /mnt/deps/vendor /workspace/project/vendor` |
| `rebar3` | `ln -s /mnt/deps/_build /workspace/project/_build` |
| `stack` | `ln -s /mnt/deps/.stack /root/.stack` |
| `deps.edn` | `ln -s /mnt/deps/.m2 /root/.m2` |
| `go` | *(no deps PVC — omit symlink)* |
| `swift-pm` | `ln -s /mnt/deps/.build /workspace/project/.build` |
| `dune` | `ln -s /mnt/deps/_build /workspace/project/_build` |
| `dub` | `ln -s /mnt/deps/.dub /root/.dub` |

Always append `chown -R 1000:1000 /workspace/project` as the last command.

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
          - ln -s /mnt/deps/{package-manager-cache} {target-path}  # omit if no deps PVC
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

### work/runs/{run-name}/substrates/{name}-onboarding.yaml

Onboarding agent entry point. PVC names here are the single source of truth — `crdtemplate` must copy them exactly.

Two distinct sets of files come from `scaffold.files`:

**All scaffold files** → `pvcs.scaffold` — the full set of files in `work/runs/{run-name}/generation/{name}/scaffold/`.

**Dependency manifest files** → `pvcs.deps.def.files` — the subset of `scaffold.files` that declare dependencies for the platform's package manager:

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

Only include files present in `scaffold.files` for this platform.

```yaml
model: substrate-onboarding
metadata:
  name: {name}
def:
  substrate: {name}
  dockerfile: work/runs/{run-name}/generation/{name}/Dockerfile
  pvcs:
    deps:
      name: substrate-{name}-deps-v1
      type: install
      def:
        files:
          - source: work/runs/{run-name}/generation/{name}/scaffold/{manifest file}
            dest: /mnt/deps/{manifest file}
        commands:
          - {dependency install command — e.g. npm install, mvn dependency:resolve}
    scaffold:
      name: substrate-{name}-scaffold-v1
      path: work/runs/{run-name}/generation/{name}/scaffold/
    plugins:
      name: substrate-{name}-plugins-v1
      plugins:
        - id: {publisher}.{extension-name}
          source: open-vsx
  manifests:
    - work/runs/{run-name}/generation/{name}/infra/pvc.yaml
```

---

## service

Generate for every platform where `service` is in `for`.

### work/runs/{run-name}/{name}/entrypoint.sh

Reads `RUNNERD_COMMAND` and maps it to the platform command. Replace `{commands[serve].run}`, `{commands[test].run}`, `{commands[build].run}` with the values from `commands` in `domains.yaml`.

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

### work/runs/{run-name}/{name}/health.sh

Checks whether the service port is accepting connections.

```bash
#!/bin/bash
. "/repo/assets/bootstrappers/lib/bootstrap.sh"
bootstrap_health_check
```

### work/runs/{run-name}/generation/{name}/Dockerfile — service, `imageStyle: image`

```dockerfile
FROM {official image}
WORKDIR /app
COPY work/runs/{run-name}/generation/{name}/scaffold/ /app/
RUN {dependency install command}
COPY work/runs/{run-name}/{name}/entrypoint.sh /bootstrapper/entrypoint.sh
COPY work/runs/{run-name}/{name}/health.sh /bootstrapper/health.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
ENTRYPOINT ["/bootstrapper/entrypoint.sh"]
```

Example for `typescript-react`:

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY work/runs/{run-name}/generation/typescript-react/scaffold/ /app/
RUN npm install
COPY work/runs/{run-name}/typescript-react/entrypoint.sh /bootstrapper/entrypoint.sh
COPY work/runs/{run-name}/typescript-react/health.sh /bootstrapper/health.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
ENTRYPOINT ["/bootstrapper/entrypoint.sh"]
```

### work/runs/{run-name}/generation/{name}/Dockerfile — service, `imageStyle: command` without `family`

The runtime must be downloaded. Two install libraries are available:

**`github.sh`** — downloads a release asset from a GitHub repository:
```sh
. /lib/github.sh
github_download {owner}/{repo} {asset-filename} [version]
```

**`uri.sh`** — downloads an asset from a direct URL:
```sh
. /lib/uri.sh
uri_download {url} [output-filename]
```

```dockerfile
FROM debian:stable-slim
COPY assets/generation/lib/ /lib/
# Check for latest: {upstream release URL}
ENV {RUNTIME}_VERSION={latest stable version}
RUN . /lib/{installLib}.sh && \
    {download and install steps}
COPY work/runs/{run-name}/{name}/entrypoint.sh /bootstrapper/entrypoint.sh
COPY work/runs/{run-name}/{name}/health.sh /bootstrapper/health.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
ENTRYPOINT ["/bootstrapper/entrypoint.sh"]
```

### work/runs/{run-name}/generation/{name}/Dockerfile — service, `imageStyle: command` with `family`

Do not generate a per-platform Dockerfile. The runtime is installed in the shared family base image at `work/runs/{run-name}/generation/images/{family}/Dockerfile`.

### work/runs/{run-name}/substrates/{name}.yaml — service

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

---

## program

Generate for every platform where `program` is in `for`.

### work/runs/{run-name}/{name}/run.sh

Accepts a file path as `$1` and runs it using the platform invocation. Replace `{io.invocation}` with the value of `io.invocation` from `domains.yaml`.

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

Example for `javascript`/`typescript` — when the invocation depends on file extension or available tools, use a `case` block:

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

### work/runs/{run-name}/generation/{name}/Dockerfile — program, `imageStyle: image`

```dockerfile
FROM {official image}
COPY work/runs/{run-name}/{name}/run.sh /bootstrapper/run.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
```

Example for `python`:

```dockerfile
FROM python:3-alpine
COPY work/runs/{run-name}/python/run.sh /bootstrapper/run.sh
COPY assets/bootstrappers/lib/bootstrap.sh /bootstrapper/lib/bootstrap.sh
RUN chmod +x /bootstrapper/*.sh
```

### work/runs/{run-name}/generation/{name}/Dockerfile — program, `imageStyle: command` without `family`

Same pattern as service `imageStyle: command` but copy `run.sh` instead of `entrypoint.sh`/`health.sh`. No `ENTRYPOINT`.

### work/runs/{run-name}/generation/{name}/Dockerfile — program, `imageStyle: command` with `family`

Do not generate a per-platform Dockerfile. The runtime is installed in the shared family base image at `work/runs/{run-name}/generation/images/{family}/Dockerfile`.

### work/runs/{run-name}/substrates/{name}.yaml — program

```yaml
model: substrate
metadata:
  name: {name}
def:
  domain:
    progLang: {domain.progLang}
  engine: container
  image: reg.jobico.local/runnerd-{name}:latest
  resources:
    memoryMiB: {resources.memoryMiB}
    cpuMillis: {resources.cpuMillis}
  io:
    extension: {io.extension}
    invocation: {io.invocation}
```

---

## `service` + `program`

Assets that apply to every platform where `service` or `program` (or both) is in `for`.

### work/runs/{run-name}/generation/images/{family}/Dockerfile

Generated once per entry in `def.families`. Builds the family base image that `imageStyle: command` platforms extend. Applies to all platforms with `imageStyle: command` and a `family` set, regardless of their `for` values.

Read `def.families` for the base image. Read all `def.domains` entries where `imageStyle: command` and `family` matches — these are the runtimes to install.

```dockerfile
# BEGIN FIXED ---------------------------------------------------------------
FROM {def.families[name].baseImage}
COPY assets/generation/lib/ /lib/
# END FIXED -----------------------------------------------------------------

# BEGIN RUNTIMES ------------------------------------------------------------
# One labeled block per platform entry with imageStyle: command and this family.
# Use assets/generation/lib/{installLib}.sh for each runtime.

# [{name}]
# Check for latest: {upstream release URL}
ENV {RUNTIME}_VERSION={latest version}
RUN . /lib/{installLib}.sh && \
    {download and install steps}

# END RUNTIMES --------------------------------------------------------------
```

Rules:
- One `# [{name}]` block per matching platform
- Deduplicate by `runtime` — if two platforms share the same runtime, one block covers both
- `COPY assets/generation/lib/` is in FIXED — never repeated per runtime
- No bootstrapper scripts, no scaffold, no entrypoint — base image only

### Bootstrapper Library Functions

| Function | Purpose |
|---|---|
| `bootstrap_cd` | Change to `RUNNERD_WORK_DIR` (default `/app/src`) |
| `bootstrap_append_args` | Set `_BOOTSTRAP_ARGS` array to `"$@"` + `RUNNERD_ARGS` |
| `bootstrap_forward_signals` | Install SIGTERM/SIGINT traps forwarding to child process group |
| `bootstrap_run_with_stdin CMD [ARGS...]` | Launch CMD, pipe `RUNNERD_STDIN` if set, set `_BOOTSTRAP_CHILD_PID` |
| `bootstrap_wait` | Wait for `_BOOTSTRAP_CHILD_PID`, exit with its status |
| `bootstrap_health_check` | Check `RUNNERD_HEALTH_PORT`, bounded to 1.8s |

### Bootstrapper Constraints

- Always `#!/bin/bash`
- Always source `/repo/assets/bootstrappers/lib/bootstrap.sh`
- Always `bootstrap_forward_signals` + `bootstrap_wait`
- Never `exec` — bootstrapper must stay alive as PID 1
- No buffering on stdout/stderr

---

## `environment-domain-ide` + `service` + `program`

### work/runs/{run-name}/generation/{name}/README.md

Documents: platform identity, `for` functions, scaffold files and their purpose, editable slot, how to build the container image manually.
