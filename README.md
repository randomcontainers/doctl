# doctl

Container images with [doctl](https://docs.digitalocean.com/reference/doctl/), the command-line interface for the DigitalOcean API. doctl creates, lists and deletes Droplets, Kubernetes clusters, managed databases, App Platform apps, domains, load balancers and the other resources of a DigitalOcean account. It is compiled from the release's source tarball with a current Go release on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when doctl publishes a release, when the pinned Go release is raised and when the base image changes. The default image adds the OpenSSH client, which `doctl compute ssh` runs.

This is an unofficial build, not affiliated with or endorsed by DigitalOcean. Report problems with the image in this repository and problems with doctl itself [upstream](https://github.com/digitalocean/doctl/issues).

## Quick start

Create a personal access token in the [control panel](https://cloud.digitalocean.com/account/api/tokens) and put it in `DIGITALOCEAN_ACCESS_TOKEN` in your shell. `-e DIGITALOCEAN_ACCESS_TOKEN` without a value passes the variable to the container:

```sh
docker run --rm -e DIGITALOCEAN_ACCESS_TOKEN ghcr.io/randomcontainers/doctl account get
docker run --rm -e DIGITALOCEAN_ACCESS_TOKEN ghcr.io/randomcontainers/doctl compute droplet list
```

A few more commands:

```sh
# Create an App Platform app from the spec in the current directory
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e DIGITALOCEAN_ACCESS_TOKEN \
  ghcr.io/randomcontainers/doctl apps create --spec app.yaml

# JSON for scripts
docker run --rm -e DIGITALOCEAN_ACCESS_TOKEN ghcr.io/randomcontainers/doctl \
  kubernetes cluster list -o json

# Chosen columns without the header line
docker run --rm -e DIGITALOCEAN_ACCESS_TOKEN ghcr.io/randomcontainers/doctl \
  compute droplet list --format Name,PublicIPv4 --no-header
```

Without arguments the image prints `doctl --help`. In CI, pin a major version tag such as `2` or a digest. doctl puts nearly every release in a new minor version, and a minor version tag such as `2.0` stays on its last build once the next minor release is built (see [Tags](#tags)).

## Tokens, contexts and settings

doctl takes the token from `--access-token` (`-t`), from `DIGITALOCEAN_ACCESS_TOKEN`, or from its config file, where `doctl auth init` saves it. The config file is `$HOME/.config/doctl/config.yaml`, and `HOME` is `/cache` in the image, so a saved token is gone with the container unless `/cache` is a volume:

```sh
docker run --rm -it -v doctl-home:/cache ghcr.io/randomcontainers/doctl auth init
docker run --rm -v doctl-home:/cache ghcr.io/randomcontainers/doctl compute droplet list
```

`auth init` asks for the token, hence `-it`. `--context <name>` keeps the tokens of several accounts or teams side by side in that file, and `DIGITALOCEAN_CONTEXT` or `doctl auth switch` selects one. `-t` and `DIGITALOCEAN_ACCESS_TOKEN` apply only to the default context: with any other context, doctl uses the token saved for that context and ignores both. To use a config file from your directory instead, pass `--config /work/doctl.yaml`, and to use another API endpoint, set `DIGITALOCEAN_API_URL`. `doctl auth login` signs in through a browser, which then redirects to a port on 127.0.0.1 inside the container. A browser on the host cannot reach it unless the container shares the host's network (`--network host` on Linux), so use a token with the image.

doctl 2 masks passwords and other secrets in its output, JSON included, unless you pass `--show` or a CI variable such as `CI=true` is set. Variables of the host do not reach the container unless you pass them, so a CI job that reads a database password from `doctl databases connection` adds `--show`, or `-e CI`.

## doctl compute ssh

`doctl compute ssh <droplet>` looks up the Droplet's address and runs `ssh`, which the default image has. Pass your ssh-agent, and add `-it` for the login session:

```sh
docker run --rm -it --user "$(id -u):$(id -g)" -e DIGITALOCEAN_ACCESS_TOKEN \
  -v "$SSH_AUTH_SOCK:/agent.sock" -e SSH_AUTH_SOCK=/agent.sock \
  ghcr.io/randomcontainers/doctl compute ssh web-1
```

On Docker Desktop, mount `/run/host-services/ssh-auth.sock` instead of `$SSH_AUTH_SOCK`. To use a key file, mount it and name it with `--ssh-key-path`, for example `-v "$HOME/.ssh:/ssh:ro"` and `--ssh-key-path /ssh/id_ed25519`. doctl logs in as `root` unless you give `--ssh-user`. ssh does not find your `~/.ssh/known_hosts` in the container, so it asks whether to trust the Droplet's host key, and the answer is lost with the container. To check the key against your own file, mount an ssh configuration that names it:

```sh
printf 'UserKnownHostsFile /ssh/known_hosts\n' > ssh_config
docker run --rm -it --user "$(id -u):$(id -g)" -e DIGITALOCEAN_ACCESS_TOKEN \
  -v "$HOME/.ssh:/ssh:ro" -v "$PWD/ssh_config:/etc/ssh/ssh_config:ro" \
  ghcr.io/randomcontainers/doctl compute ssh web-1 --ssh-key-path /ssh/id_ed25519
```

`ssh` works for any UID given with `--user`; [randomcontainers/openssh-client](https://github.com/randomcontainers/openssh-client) explains how.

## Commands that need more than the image has

- `doctl kubernetes cluster kubeconfig save` writes credentials that run `doctl` whenever `kubectl` needs a new token, so they work only where doctl is installed. With `--expiry-seconds`, it writes a token that expires instead. The image has no `kubectl`. This writes the kubeconfig to your directory:

  ```sh
  docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e KUBECONFIG=/work/kubeconfig \
    -e DIGITALOCEAN_ACCESS_TOKEN ghcr.io/randomcontainers/doctl \
    kubernetes cluster kubeconfig save my-cluster --expiry-seconds 3600
  ```

- `doctl registry login` writes Docker credentials to `/cache/.docker/config.json`, or to `config.json` in `DOCKER_CONFIG`. They are of use only when you mount that directory for a Docker client outside the container.
- `doctl apps dev build` needs a Docker engine.
- `doctl serverless install` downloads Node.js from nodejs.org and the serverless plugin into `/cache/.config/doctl/sandbox`. Those Node.js builds need glibc, so `doctl serverless` does not run on the Alpine images.
- `doctl version` asks api.github.com for the newest release, to tell you about an update. Without network access it prints the version after the lookup fails.

## What is in the image

| | slim | default |
|---|---|---|
| `doctl` in `/usr/local/bin`, built without cgo, so it is statically linked and loads no libraries | yes | yes |
| The OpenSSH client (`ssh`, `scp`, `ssh-keygen` and the other client tools), for `doctl compute ssh` | no | yes |

Not included: `kubectl`, Docker, Node.js and Git. The OpenSSH client is the [randomcontainers/openssh-client](https://github.com/randomcontainers/openssh-client) build.

The Go version, the SHA-256 of the source tarball, the commit it was made from and every Go module with its version and checksum are in `/usr/local/share/randomcontainers/doctl/buildinfo`.

## Default or slim

Use the default image (`latest`) for general work: `doctl compute ssh` needs `ssh`. `slim` has `doctl` alone. Use it as the base for your own image, or for scripts that only call the API.

The default image is also published as `ghcr.io/randomcontainers/doctl-openssh-client`, built in the [doctl-openssh-client](https://github.com/randomcontainers/doctl-openssh-client) repository with the same contents and a different digest.

## Tags

`<version>` is a doctl release such as `2.0.0`. `<minor>` and `<major>` are its shorter forms, `2.0` and `2`, and follow the newest release in that series. Only the newest release is built, so `<minor>` stops moving when the next minor release comes out.

| Default (with the OpenSSH client) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current doctl version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is `/cache` for every user, and `/cache` can be written by anyone. `USER` is set to `doctl`, because doctl looks up the current user when it starts and, for a UID that has no entry in `/etc/passwd`, fails unless `USER` is set. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

doctl keeps its config file and caches in `/cache/.config/doctl/`, and `kubeconfig save` writes `/cache/.kube/config` unless `KUBECONFIG` names another file. Files that commands such as `apps create --spec` read are relative to `/work`.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new doctl release and are rebuilt when the base image changes. Switch to root to install more, then back. This adds `jq`, for reading the output of `doctl ... -o json` in a script:

```dockerfile
FROM ghcr.io/randomcontainers/doctl:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends jq \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache jq`. The entrypoint is `["tini", "--", "doctl"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new doctl releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/doctl/` holds the version, the source links, the build details, the license files, the MPL-2.0 module sources and `runtime-deps`, the list of distro packages `doctl` needs at run time beyond those in the base image (none).

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/doctl:latest \
  --repo randomcontainers/doctl --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/doctl-openssh-client` are built in that repository, so verify them with `--repo randomcontainers/doctl-openssh-client` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/doctl:latest --format '{{ json .SBOM }}'
```

doctl attaches a source tarball, `doctl-<version>-source.tar.gz`, to each release; it is a git archive of the release tag. The SHA-256 in `package.yml` is checked against the release's checksums file, `doctl-<version>-checksums.sha256`, when a version is picked up, and the build checks the tarball against it before compiling. The Go toolchain is downloaded from Go's release server (dl.google.com, as linked from go.dev/dl) and checked against the SHA-256 in the Dockerfile. The tarball also contains the Go modules in `vendor/`; the build does not use them and downloads the modules from the Go module proxy instead, checked against doctl's `go.sum`. `buildinfo` records the commit named in the tarball's header and the Go modules.

Image scanners read the list of Go modules from the `doctl` binary. doctl's `go.mod` pins old versions of several modules, among them Docker, containerd, BuildKit, `golang.org/x/crypto` and `golang.org/x/net`, so scanners report advisories against them. Many of those advisories concern packages that doctl does not compile in, such as the Docker daemon, containerd's CRI plugin and BuildKit's builder. Others concern packages it does use, such as `golang.org/x/crypto/ssh` and `golang.org/x/net/html`, and go away when doctl raises those modules. The images use the module versions in doctl's `go.mod` and change only the Go release.

## Updates

The project checks the releases of [digitalocean/doctl](https://github.com/digitalocean/doctl/releases) every 15 minutes. A release is picked up once it is 24 hours old; pre-releases are skipped. Its source tarball is checked against the release's checksums file, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

doctl's own binaries keep the Go release they were built with until the next doctl release. These images are compiled with the Go release pinned in the Dockerfiles instead, which is raised by hand when Go publishes a security release, and each raise rebuilds the images of the current version. A doctl release that needs a newer Go than the pin does not build until the pin is raised.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new OpenSSH client image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t doctl:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of packages Go builds in parallel. The build downloads a few hundred megabytes of Go modules. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

doctl is released under the [Apache License 2.0](https://github.com/digitalocean/doctl/blob/main/LICENSE.txt). The binary also contains:

- the Go standard library (BSD-3-Clause);
- the Go modules doctl depends on, under Apache-2.0, MIT, BSD-3-Clause, BSD-2-Clause, ISC and MPL-2.0, the last for HashiCorp's errwrap, go-cleanhttp, go-multierror, go-retryablehttp and hcl;
- the Liberation Mono font (OFL-1.1), which chroma embeds for its SVG output.

The image's license label is `Apache-2.0 AND MIT AND BSD-3-Clause AND BSD-2-Clause AND MPL-2.0 AND ISC AND OFL-1.1`. The license files are in `/usr/local/share/randomcontainers/doctl/licenses/`, with one directory per Go module. The font's license and copyright notices are in chroma's directory: in `COPYING` for chroma 2.24.0 and later, and in `formatters/svg/OFL-1.1.txt` for older releases, which have them only at the top of the font's Go source file. The directory of `github.com/opencontainers/go-digest` also has its `LICENSE.docs` (CC-BY-SA-4.0), which covers go-digest's documentation. The documentation is not compiled into `doctl`, so CC-BY-SA-4.0 is not in the label.

MPL-2.0 requires whoever distributes a binary built from MPL-2.0 files to make their source available to its recipients. The source zips of those modules, downloaded from the Go module proxy and checked against `go.sum`, are in `/usr/local/share/randomcontainers/doctl/mpl-sources/`, and `/usr/local/share/randomcontainers/doctl/source` lists them after the link to doctl's source tarball.

The default image adds the OpenSSH client (`SSH-OpenSSH AND MIT`); see [randomcontainers/openssh-client](https://github.com/randomcontainers/openssh-client#licenses) for its sources and license files. The packages from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
