# assets/pipeline/lib — Install Libraries

Shell libraries used in generated Dockerfiles to install runtimes that have no official Docker Hub image (`imageStyle: command`). Source the library, then call its function.

All libraries are copied into the image at `/lib/` at the top of the Dockerfile:

```dockerfile
COPY assets/pipeline/lib/ /lib/
```

---

## github.sh

Downloads a release asset from a GitHub repository.

```sh
. /lib/github.sh
github_download {owner}/{repo} {asset-filename} [version]
```

- `owner/repo` — GitHub repository (e.g. `JetBrains/kotlin`)
- `asset-filename` — exact filename of the release asset to download
- `version` — optional release tag (e.g. `2.0.0`). Tries `v{version}` first, then `{version}`. Defaults to latest release.
- Downloads the asset to the current working directory.
- Set `GITHUB_TOKEN` to avoid rate limits (60 → 5000 req/hr).

### Examples

**Kotlin compiler:**
```dockerfile
ENV KOTLIN_VERSION=2.0.0
RUN . /lib/github.sh && \
    github_download JetBrains/kotlin kotlin-compiler-${KOTLIN_VERSION}.zip ${KOTLIN_VERSION} && \
    unzip kotlin-compiler-${KOTLIN_VERSION}.zip -d /usr/local && \
    rm kotlin-compiler-${KOTLIN_VERSION}.zip && \
    ln -s /usr/local/kotlinc/bin/kotlinc /usr/local/bin/kotlinc && \
    ln -s /usr/local/kotlinc/bin/kotlin /usr/local/bin/kotlin
```

**Clojure CLI:**
```dockerfile
ENV CLOJURE_VERSION=1.11.1.1435
RUN . /lib/github.sh && \
    github_download clojure/brew-install clojure-tools-${CLOJURE_VERSION}.tar.gz ${CLOJURE_VERSION} && \
    tar xzf clojure-tools-${CLOJURE_VERSION}.tar.gz && \
    ./clojure-tools/install && \
    rm -rf clojure-tools-${CLOJURE_VERSION}.tar.gz clojure-tools
```

**Groovy:**
```dockerfile
ENV GROOVY_VERSION=4.0.15
RUN . /lib/github.sh && \
    github_download apache/groovy apache-groovy-binary-${GROOVY_VERSION}.zip ${GROOVY_VERSION} && \
    unzip apache-groovy-binary-${GROOVY_VERSION}.zip -d /usr/local && \
    rm apache-groovy-binary-${GROOVY_VERSION}.zip && \
    ln -s /usr/local/groovy-${GROOVY_VERSION} /usr/local/groovy
ENV PATH=/usr/local/groovy/bin:$PATH
```

**Elixir:**
```dockerfile
ENV ELIXIR_VERSION=1.16.0
RUN . /lib/github.sh && \
    github_download elixir-lang/elixir elixir-otp-26.zip ${ELIXIR_VERSION} && \
    unzip elixir-otp-26.zip -d /usr/local/elixir && \
    rm elixir-otp-26.zip
ENV PATH=/usr/local/elixir/bin:$PATH
```

---

## uri.sh

Downloads an asset from a direct URI.

```sh
. /lib/uri.sh
uri_download {url} [output-filename]
```

- `url` — full download URL
- `output-filename` — optional. Derived from the URL if omitted.
- Downloads the asset to the current working directory.

### Examples

**sbt (Scala build tool):**
```dockerfile
ENV SBT_VERSION=1.9.9
RUN . /lib/uri.sh && \
    uri_download https://github.com/sbt/sbt/releases/download/v${SBT_VERSION}/sbt-${SBT_VERSION}.tgz && \
    tar xzf sbt-${SBT_VERSION}.tgz -C /usr/local && \
    rm sbt-${SBT_VERSION}.tgz
ENV PATH=/usr/local/sbt/bin:$PATH
```

**GHCup (Haskell toolchain installer):**
```dockerfile
RUN . /lib/uri.sh && \
    uri_download https://get-ghcup.haskell.org ghcup-install.sh && \
    chmod +x ghcup-install.sh && \
    BOOTSTRAP_HASKELL_NONINTERACTIVE=1 \
    BOOTSTRAP_HASKELL_INSTALL_STACK=1 \
    BOOTSTRAP_HASKELL_INSTALL_HLS=0 \
    ./ghcup-install.sh && \
    rm ghcup-install.sh
ENV PATH=/root/.ghcup/bin:/root/.cabal/bin:$PATH
```
