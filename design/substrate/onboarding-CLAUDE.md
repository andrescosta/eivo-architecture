# onboarding/CLAUDE.md

## Priority

Each platform in `def.domains` has a `priority` field (0, 1, 2, …). No more than 3 platforms share the same priority value.

The agent processes platforms in priority order — all platforms at priority 0 first, then priority 1, and so on. Within a priority group, platforms can be onboarded in parallel or sequentially. The driver prompt specifies which priority groups to run.

This allows incremental validation: onboard the first small batch, verify manually, then proceed to the next group.

## What This Does

Validates generated platform files and provisions the platform infrastructure in the cluster. Reads `work/runs/{run-name}/substrates/{name}-onboarding.yaml` as the entry point for each platform.

Two sequential phases — Phase 2 runs only after Phase 1 passes. If Phase 2 fails, fix the generated files and retry from Phase 1.

## Onboarding Container

The agent runs inside a container built from `Dockerfile` at the repo root (the bootstrap Dockerfile), launched via `run-agent.sh`. The repo is mounted at `/repo` and `work/` at `/work` — nothing is copied into the container.

## Source

`work/runs/{run-name}/substrates/{name}-onboarding.yaml` is the entry point. It references all other generated files:

```yaml
model: substrate-onboarding
metadata:
  name: typescript-react
def:
  substrate: typescript-react
  dockerfile: work/runs/{run-name}/generation/typescript-react/Dockerfile
  pvcs:
    deps:
      name: substrate-typescript-react-deps-v1
      type: install
      def:
        files:
          - source: work/runs/{run-name}/generation/typescript-react/scaffold/package.json
            dest: /mnt/deps/package.json
        commands:
          - npm install
    scaffold:
      name: substrate-typescript-react-scaffold-v1
      path: work/runs/{run-name}/generation/typescript-react/scaffold/
    plugins:
      name: substrate-typescript-react-plugins-v1
      plugins:
        - id: vscode.typescript-language-features
          source: vscode
        - id: dbaeumer.vscode-eslint
          source: open-vsx
  manifests:
    - work/runs/{run-name}/generation/typescript-react/infra/pvc.yaml
```

## Phase 1: Validate and Fix *(log: `validate`)*

Validate every referenced file before attempting any provisioning. Fix issues directly in the generated files. Iterate until all checks pass. Write `validate: done` to the log only when all checks pass.

**Agent-automatable:**

- **Dockerfile** — syntax is valid (`docker build --check`), `FROM` image exists and is pullable, bootstrapper paths reference existing files in `work/runs/{run-name}/{name}/`
- **Bash syntax** — `bash -n` on all bootstrapper scripts (`entrypoint.sh`, `health.sh`, `run.sh`)
- **work/runs/{run-name}/generation/{name}/scaffold/** — all files listed in `pvcs.scaffold.path` exist, content is syntactically valid for the file type
- **work/runs/{run-name}/generation/{name}/infra/pvc.yaml** — valid Kubernetes YAML, PVC names match exactly those declared in `pvcs.*.name`
- **work/runs/{run-name}/substrates/{name}.yaml** — valid ADL YAML, all required fields present (`domain`, `engine`, `image`, `commands`)
- **work/runs/{run-name}/substrates/{name}-crdtemplate.yaml** — valid ADL YAML, PVC `claimName` values match exactly those in `substrate-onboarding`
- **Plugin IDs** — each plugin in `pvcs.plugins.plugins` exists on Open VSX (`https://open-vsx.org/api/{publisher}/{name}`)

**Requires manual validation (outside agent scope):**

- crdtemplate correctness end-to-end — requires the environment operator running in the cluster
- PVC population result — whether installed deps are complete and correct for the platform
- Whether the platform boots successfully using the populated PVCs

## Phase 2: Provision

Execute in this order:

1. **Build family base images** *(log: `build_family`)* — for each `work/runs/{run-name}/generation/images/{family}/Dockerfile`, check if `reg.jobico.local/eivo-{family}:latest` already exists in the registry. Build and push only if it does not.
2. **Build platform image** *(log: `build`)* — `docker build -f work/runs/{run-name}/generation/{name}/Dockerfile .` from repo root, push to `reg.jobico.local/runnerd-{name}:latest`
3. **Test bootstrappers** *(log: `test`)* — run a container from the built platform image and verify the bootstrapper scripts work correctly inside it:
   - `RUNNERD_COMMAND=serve entrypoint.sh` — starts and keeps running
   - `RUNNERD_COMMAND=test entrypoint.sh` — exits with correct exit code
   - `run.sh /path/to/code` — runs file and exits with correct exit code (runner platforms only)
   - `health.sh` — returns 0 only when port is listening, within 2 seconds
   - SIGTERM — terminates entire child process tree cleanly
   If any test fails, fix the bootstrapper in `work/runs/{run-name}/{name}/`, rebuild the image, and retest before continuing.
4. **Download plugins** *(log: `plugins`)* — fetch each plugin `.vsix` from Open VSX into a local staging directory
5. **Apply PVC manifests** *(log: `apply_pvcs`)* — `kubectl apply -f {manifests}`
6. **Populate scaffold PVC** *(log: `scaffold_pvc`)* — copy files from `pvcs.scaffold.path` into the scaffold PVC
7. **Populate deps PVC** *(log: `deps_pvc`)* — copy `pvcs.deps.def.files` into the PVC mount, run `pvcs.deps.def.commands` inside the mount, verify the install command exits 0 and expected top-level items are present (e.g. `node_modules/`)
8. **Populate plugins PVC** *(log: `plugins_pvc`)* — copy downloaded `.vsix` files into the plugins PVC

## Constraints

- Never modify `work/runs/{run-name}/substrates/{name}-onboarding.yaml` — it is the source of truth
- Fixes go into the generated files it references (`Dockerfile`, `work/runs/{run-name}/generation/{name}/scaffold/`, `work/runs/{run-name}/generation/{name}/infra/pvc.yaml`, `work/runs/{run-name}/substrates/{name}.yaml`, `work/runs/{run-name}/substrates/{name}-crdtemplate.yaml`)
- Phase 2 does not start until Phase 1 passes completely
- If any Phase 2 step fails, diagnose, fix the generated files, and restart from Phase 1
