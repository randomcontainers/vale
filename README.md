# vale

Container images with [Vale](https://vale.sh/), a command-line linter for prose. Vale checks Markdown, MDX, HTML and other markup, and the comments in source code, against the style rules you configure in `.vale.ini`. The images contain no styles. Vale is compiled from its release tag on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when Vale publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the Vale project. Report problems with the image in this repository and problems with Vale itself [upstream](https://github.com/vale-cli/vale/issues).

## Quick start

Vale reads `.vale.ini` from the working directory or the nearest parent directory that has one. This configuration uses Vale's built-in rules and the Google style from Vale's package library:

```ini
StylesPath = styles
MinAlertLevel = suggestion
Packages = Google

[*.md]
BasedOnStyles = Vale, Google
```

Download the packages into `styles/`, which needs network access, then check the Markdown files in `docs/`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/vale sync
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/vale docs/
```

Vale exits with 1 when it reports an alert at the `error` level and with 2 when it cannot run, for example because of a mistake in `.vale.ini`. A few more options:

```sh
# One alert per line, as file:line:column:check:message
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/vale \
  --output=line docs/

# Alerts as JSON, without failing on errors
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/vale \
  --output=JSON --no-exit docs/

# Show only errors
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/vale \
  --minAlertLevel=error docs/
```

Without arguments the image prints `vale --help`. In CI, pin a major version tag such as `3` or a digest, and either run `vale sync` first or commit the downloaded styles to the repository so that linting needs no network.

## Styles and configuration

Set `StylesPath` in `.vale.ini`. A relative path is relative to the file, so `vale sync` writes the packages into your project and later runs find them there. Without it, Vale keeps styles in the home directory, which the container cannot write to when it runs with `--user`.

Your own rules go into a directory of `StylesPath` named after the style, one YAML file per rule, and are enabled with `BasedOnStyles`. Vale's documentation describes [`.vale.ini`](https://vale.sh/docs/vale-ini) and [styles](https://vale.sh/docs/styles).

## File formats

Vale reads these formats itself, without other programs: Markdown, MDX (no Node.js needed), MyST, Quarto, HTML, Org, Jupyter notebooks and plain text. In source code it checks comments and docstrings, which it finds with tree-sitter grammars compiled into the binary for about twenty languages, among them C and C++, Go, Java, JavaScript and TypeScript, Python, Ruby and Rust.

A few formats need a converter that the image does not have: reStructuredText needs `rst2html` from Docutils, AsciiDoc needs `asciidoctor`, XML needs `xsltproc`, DITA needs `dita` and Typst needs `typst2vast`. See [Extending the slim image](#extending-the-slim-image) for adding one.

## What is in the image

- `vale` in `/usr/local/bin`, built with cgo for the tree-sitter grammars. It links only the distro's C and C++ runtime libraries.
- The built-in `Vale` style and its spelling dictionary for American English.

Not included: styles such as Google, Microsoft, write-good or proselint, and the converters listed under [File formats](#file-formats). The Go version, the commit and every Go module with its version and checksum are in `/usr/local/share/randomcontainers/vale/buildinfo`.

## Default or slim

Vale's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a Vale release such as `3.24.0`. `<minor>` and `<major>` are its shorter forms, `3.24` and `3`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Vale version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Linting writes nothing to the working directory. `vale sync` writes into `StylesPath`, so the files it downloads belong to the user the container runs as.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Vale release and are rebuilt when the base image changes. Switch to root to install more, then back. This adds Docutils, so that Vale can check reStructuredText:

```dockerfile
FROM ghcr.io/randomcontainers/vale:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends python3-docutils \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache py3-docutils`. Both packages put `rst2html` on `PATH`, where Vale looks for it. The entrypoint is `["tini", "--", "vale"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new Vale releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/vale/` holds the version, the source URLs, the build details, the license files and `runtime-deps`, the list of distro packages Vale needs at run time beyond those in the base image.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/vale:latest \
  --repo randomcontainers/vale --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/vale:latest --format '{{ json .SBOM }}'
```

Vale publishes prebuilt binaries but no source tarball, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [vale-cli/vale](https://github.com/vale-cli/vale) and stops unless the tag points to the commit recorded in `package.yml`. The Go toolchain is downloaded from Go's release server (dl.google.com, as linked from go.dev/dl) and checked against the SHA-256 in the Dockerfile, and the Go modules are checked against Vale's `go.sum`. The binary records the commit it was built from, and `buildinfo` lists it with the Go modules.

## Updates

The project checks the releases of [vale-cli/vale](https://github.com/vale-cli/vale/releases) every 15 minutes. A release is picked up once it is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The Go toolchain is pinned in the Dockerfiles and raised by hand when Go publishes a security release. A Vale release that needs a newer Go than the pin does not build until the pin is raised.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t vale:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of packages Go builds in parallel. Most of the build time goes into compiling the tree-sitter grammars.

## Licenses

Vale is released under the [MIT license](https://github.com/vale-cli/vale/blob/v3/LICENSE). Its spell checker is derived from gospell (MIT), its American English dictionary comes from [en_US-web](https://github.com/vale-cli/en_US-web) (MIT), and one of its files carries the MIT notice of Write.as. The binary also contains:

- the Go standard library (BSD-3-Clause);
- the Go modules Vale depends on, under MIT, BSD-3-Clause, BSD-2-Clause and Apache-2.0;
- the tree-sitter runtime and grammars, which are MIT except the Elixir grammar (Apache-2.0); the runtime includes ICU's UTF-8 and UTF-16 macros (Unicode-DFS-2016).

The image's license label is `MIT AND BSD-3-Clause AND BSD-2-Clause AND Apache-2.0 AND Unicode-DFS-2016`. The license files are in `/usr/local/share/randomcontainers/vale/licenses/`, with one directory per Go module. The Go modules that bundle tree-sitter do not include the licenses of the runtime and the grammars, and Vale's repository does not include the license of en_US-web, so this repository keeps them in [licenses](licenses) and the build copies them into the image. The Lua grammar's repository has no license file; its Cargo.toml declares MIT and its package.json ISC. Links to the source are in `/usr/local/share/randomcontainers/vale/source`. libstdc++ and the other libraries from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
