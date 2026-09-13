# agent/CLAUDE.md

## What This Does

Generates `Dockerfile` at the repo root — the bootstrap Dockerfile that builds the agent container. This container has all tools needed to run the three stages: agent, generation, onboarding.

Run this stage once, then rebuild whenever `def.runtimes` changes.

## Source

Read `domains.yaml` → `def.runtimes`. Each entry needs a block in the RUNTIMES section. Runtimes already covered by the FIXED section do not get a block.

Entries with a `members` field declare a group: the entry itself is the base runtime (installs the shared dependency — JDK, Erlang, dotnet), and each member extends it with only the member-specific toolchain. Members do not get their own top-level block — they are nested inside their base block.

```yaml
runtimes:
  - name: java
    members: [kotlin, scala, groovy, clojure]  # base installs JDK; members add only their toolchain
  - name: erlang
    members: [elixir]
  - name: csharp
    members: [fsharp]
```

## Rules

**Skip what FIXED already provides** — `bash`, `curl`, `wget`, `git`, `nodejs`, `npm`, `build-essential` (which includes `gcc` and `g++`) are already present. Never reinstall them. `shell` (`bash`) is always available — it is not in `def.runtimes`.

**One block per runtime** — each runtime, and each of its members, gets its own labeled `# [{name}]` block with its own `RUN`. Do not consolidate apt installs across unrelated runtimes — it makes the Dockerfile hard to read and maintain. This applies within groups too: an apt-only member still gets its own block, it just skips reinstalling the shared dependency its base already installed.

**Respect runtime groups** — when a runtime has `members`, its own block installs the shared base (JDK, Erlang, dotnet). Each member's block comes right after the base block and installs only what that member adds on top — never the shared base again.

**Prefer direct downloads over interactive installers** — sdkman, asdf, and similar tools require interactive shells and are fragile in Dockerfiles. Use direct downloads (GitHub releases, vendor URLs) instead.

**Use `bash` for piped installers** — scripts piped to a shell (rustup, ghcup, dotnet-install, etc.) must use `bash`, not `sh`.

**Use latest stable versions** — for direct downloads, fetch the latest stable release at generation time. Do not hardcode old versions.

## Output

`Dockerfile` at the repo root. Once generated, build and push the image:

```bash
./build-agent.sh
```

This builds the image tagged as `eivo-substrates-agent` and pushes it to the local registry. The agent container used by `run-agent.sh` depends on this image being up to date. Rebuild whenever `def.runtimes` changes.

## Structure

The Dockerfile has two clearly marked sections:

```dockerfile
# BEGIN FIXED ---------------------------------------------------------------
FROM debian:stable-slim

RUN apt-get update && apt-get install -y \
    curl wget git ca-certificates docker.io build-essential nodejs npm \
    && rm -rf /var/lib/apt/lists/*

RUN curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
    && install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

ENV NPM_CONFIG_PREFIX=/home/node/.npm-global
ENV PATH=$NPM_CONFIG_PREFIX/bin:$PATH
RUN mkdir -p /home/node/.npm-global && chmod -R 777 /home/node/.npm-global
RUN npm install -g @anthropics/claude-code
# END FIXED -----------------------------------------------------------------

# BEGIN RUNTIMES ------------------------------------------------------------
# One labeled block per entry in def.runtimes.
# Runtimes not in def.runtimes are already covered by FIXED — skip them.
# All RUN commands execute as root.

# [cobol]
RUN apt-get update && apt-get install -y gnucobol && rm -rf /var/lib/apt/lists/*

# [java] — base (shared JDK for kotlin, scala, groovy, clojure)
RUN apt-get update && apt-get install -y default-jdk maven && rm -rf /var/lib/apt/lists/*

# [groovy] — member of java (apt package; JDK already present)
RUN apt-get update && apt-get install -y groovy && rm -rf /var/lib/apt/lists/*

# [rust] (rustup always installs latest stable)
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | bash -s -- -y --no-modify-path
ENV PATH="/root/.cargo/bin:${PATH}"

# END RUNTIMES --------------------------------------------------------------

USER node
```

**FIXED section** — always the same, never modified:
- Base image (`debian:stable-slim`)
- Basic tools + docker + kubectl
- Node.js + Claude Code (agent runtime)

**RUNTIMES section** — one labeled `# [{name}]` block per entry in `def.runtimes` (and one per member, immediately after its base). Each block installs the toolchain for that runtime and nothing else.

## Version Resolution Examples

For runtimes that require direct downloads, always resolve the latest stable version dynamically at build time. Do not hardcode version numbers.

**GitHub API — latest release tag:**
```dockerfile
# [kotlin] — member of java
RUN apt-get update && apt-get install -y unzip && rm -rf /var/lib/apt/lists/*
RUN KOTLIN_VERSION=$(curl -s https://api.github.com/repos/JetBrains/kotlin/releases/latest \
        | grep -oP '"tag_name":\s*"v\K[^"]+') \
    && curl -fL -o kotlin-compiler.zip \
        "https://github.com/JetBrains/kotlin/releases/download/v${KOTLIN_VERSION}/kotlin-compiler-${KOTLIN_VERSION}.zip" \
    && unzip -q kotlin-compiler.zip -d /opt \
    && rm kotlin-compiler.zip
ENV PATH="/opt/kotlinc/bin:${PATH}"
```

**GitHub API — latest release with filename pattern:**
```dockerfile
# [zig]
RUN ZIG_VERSION=$(curl -s https://api.github.com/repos/ziglang/zig/releases/latest \
        | grep -oP '"tag_name":\s*"\K[^"]+') \
    && curl -fL -o zig.tar.xz \
        "https://ziglang.org/download/${ZIG_VERSION}/zig-linux-x86_64-${ZIG_VERSION}.tar.xz" \
    && tar xf zig.tar.xz \
    && mv "zig-linux-x86_64-${ZIG_VERSION}" /usr/local/zig \
    && rm zig.tar.xz
ENV PATH="/usr/local/zig:${PATH}"
```

**Official installer that always resolves latest:**
```dockerfile
# [clojure] — member of java
RUN apt-get update && apt-get install -y rlwrap && rm -rf /var/lib/apt/lists/*
RUN curl -O https://download.clojure.org/install/linux-install.sh \
    && chmod +x linux-install.sh \
    && ./linux-install.sh \
    && rm linux-install.sh
```

**Piped installer — always use `bash`, not `sh`:**
```dockerfile
# [rust]
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | bash -s -- -y --no-modify-path
ENV PATH="/root/.cargo/bin:${PATH}"
```

## Constraints

- `USER node` is always last — after all `RUN` install commands (which require root)
- No `COPY` — repo and `work/` are mounted at runtime
- FIXED section is never modified
- Only runtimes in `def.runtimes` get a RUNTIMES block
