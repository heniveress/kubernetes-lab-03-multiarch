# Microservices and Kubernetes: Lab 3 - Multi-architecture Docker builds

In Lab 1 you packaged a single service and pushed it to a registry.

That image was almost certainly built for one CPU architecture only, whatever your laptop happens to be (on Windows that is usually `linux/amd64`, on Mac `linux/arm64`).

Modern deployments increasingly run on **ARM64**: Apple Silicon, AWS Graviton, Ampere, and Raspberry Pi nodes all want an `arm64` image. An `amd64`-only image will simply refuse to start there.

This lab shows **three different ways** to produce images for more than one architecture with Docker, using two tiny sample apps from the [`examples/`](examples) folder. Every example builds the image in GitHub Actions and pushes it to **Docker Hub**.

| #   | Example workflow                                                             | Source                                                                    | Strategy                                                                                | Published image tag       |
| --- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------- |
| 1   | [`python-fastapi-multiarch`](.github/workflows/python-fastapi-multiarch.yml) | [`examples/python-fastapi-multiarch/`](examples/python-fastapi-multiarch) | ARM64 only, **native** build on an ARM64 runner                                         | `python-arm64`            |
| 2   | [`csharp-aspnet`](.github/workflows/csharp-aspnet.yml)                       | [`examples/csharp-aspnet/`](examples/csharp-aspnet)                       | AMD64 + ARM64, each built **natively** on its own runner, then merged into one manifest | `csharp-multiarch-native` |
| 3   | [`pastebin-csharp-qemu`](.github/workflows/pastebin-csharp-qemu.yml)         | [`examples/csharp-aspnet/`](examples/csharp-aspnet)                       | AMD64 + ARM64 on **one** runner using QEMU emulation                                    | `csharp-multiarch-qemu`   |

All three tags live in a single Docker Hub repository you configure, for example:

```text
darktohka/multiarch-lab:python-arm64
darktohka/multiarch-lab:csharp-multiarch-native
darktohka/multiarch-lab:csharp-multiarch-qemu
```

> The two C# examples build the **exact same source**. Only the build method
> differs, so you can compare the trade-offs directly.

---

## Background: what "multi-arch" actually means

A Docker image is not one blob. A tag points at a **manifest list** (also called an **OCI image index**), which lists one **manifest per platform** (`os`/`architecture` pair). When you `docker pull` a tag, the client reads the index, picks the entry that matches your machine, and downloads only that image.

```text
darktohka/multiarch-lab:csharp-multiarch-qemu   (tag)
        │
        ▼
   manifest list / image index
        ├── linux/amd64  ──► manifest ──► layers for x86_64
        └── linux/arm64  ──► manifest ──► layers for aarch64
```

This is why a single tag like `nginx:latest` works on your laptop and on a Graviton server at the same time. The job of a multi-arch build is to produce that index.

There are two ways to produce the per-architecture images:

- **Natively** - run the build on a machine of that architecture. Fast, but you
  need (or rent) a machine of every architecture you target.
- **Emulated (QEMU)** - run the build on one machine of one architecture and
  use [`binfmt_misc`](https://en.wikipedia.org/wiki/Binfmt_misc) + QEMU to
  execute foreign-architecture binaries. One machine is enough, but the emulated
  build is much slower.

The three examples below are, in order: one native architecture, all
architectures native, and all architectures emulated.

---

## Repository layout

```text
.
├── .github/workflows/
│   ├── python-fastapi-multiarch.yml   # Example 1: native ARM64 only
│   ├── csharp-aspnet.yml              # Example 2: native AMD64 + ARM64, merged
│   └── pastebin-csharp-qemu.yml       # Example 3: QEMU AMD64 + ARM64
├── examples/
│   ├── python-fastapi-multiarch/      # Python 3.14 + FastAPI mini-pastebin
│   ├── csharp-aspnet/                 # C# / .NET 8 ASP.NET Core mini-pastebin
│   └── docs/API.md                    # shared API contract
└── README.md
```

Both apps expose the same tiny HTTP API (see [`examples/docs/API.md`](examples/docs/API.md)),
so the differences you observe come only from the packaging and build method.

---

## Prerequisites

- A **Docker Hub** account.
- A **GitHub** account, with this repository forked (or copied) into your own
  account, and **Actions enabled**.
- Nothing else - all builds happen on GitHub's runners. You do not need a local
  ARM machine.

---

## Setup - step by step

### 1. Fork the repository

Fork this repository into your own GitHub account. All commands below assume you work on your fork.

### 2. Create a Docker Hub access token

Do **not** put your Docker Hub password in a secret. Create a scoped token:

1. Log in at <https://hub.docker.com/>.
2. Open **Account settings → Personal access tokens → Generate new token**.
3. Description: `github-actions-multiarch`. Access permissions: **Read & Write**.
4. Copy the token - Docker Hub only shows it once.

### 3. Configure the `DOCKER_IMAGE` variable

The workflows read the image repository from a repository **variable**:

1. In your fork, go to **Settings → Secrets and variables → Actions**.
2. Open the **Variables** tab → **New repository variable**.
3. Name: `DOCKER_IMAGE`
4. Value: `<your-dockerhub-username>/multiarch-lab` - for example
   `darktohka/multiarch-lab`.
   The part before the `/` must be your Docker Hub username; the workflows
   derive the login username from it.

### 4. Configure the `DOCKERHUB_TOKEN` secret

1. On the same page, open the **Secrets** tab → **New repository secret**.
2. Name: `DOCKERHUB_TOKEN`
3. Value: the token you created in step 2.
4. **Add secret**.

### 5. (Optional) pre-create the Docker Hub repository

Docker Hub usually creates the repository on the first successful push. If you
prefer, create `multiarch-lab` manually at
<https://hub.docker.com/repository/create> and set it to **Public**.

### 6. Run the workflows

Each example is a separate workflow so you can run them independently:

- **Trigger by pushing:** any commit will cause a rebuild.
- **Trigger manually:** **Actions →** pick a workflow **→ Run workflow**.

Run all three once so every tag exists.

---

## Verifying the result

`docker buildx imagetools inspect` prints the manifest list - this is the
quickest way to prove which platforms an image ships.

**Example 1 - ARM64 only** (`python-arm64`):

```console
$ docker buildx imagetools inspect darktohka/multiarch-lab:python-arm64
Name:      docker.io/darktohka/multiarch-lab:python-arm64
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:...
Manifests:
  Name:        docker.io/darktohka/multiarch-lab:python-arm64@sha256:...
  Platform:    linux/arm64
```

Exactly one platform: `linux/arm64`.

**Examples 2 & 3 - AMD64 + ARM64** (`csharp-multiarch-native`,
`csharp-multiarch-qemu`):

```console
$ docker buildx imagetools inspect darktohka/multiarch-lab:csharp-multiarch-native
Name:      docker.io/darktohka/multiarch-lab:csharp-multiarch-native
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:...
Manifests:
  Name:        ...@sha256:...
  Platform:    linux/amd64
  Name:        ...@sha256:...
  Platform:    linux/arm64
```

Two platforms: `linux/amd64` **and** `linux/arm64`.

You can also check from the command line that the image really runs on the
target architecture. `--platform` forces Docker to fetch and run the ARM64
entry (locally this needs QEMU/binfmt, which Docker Desktop provides out of the
box):

```console
$ docker run --rm --platform linux/arm64 \
    darktohka/multiarch-lab:csharp-multiarch-qemu uname -m
aarch64

$ docker run --rm --platform linux/amd64 \
    darktohka/multiarch-lab:csharp-multiarch-qemu uname -m
x86_64
```

`uname -m` printing `aarch64` proves you are running the ARM64 image.

---

## Running the sample apps locally

Each example folder has a `docker-compose.yml` for local development. Compose
builds for your host architecture only, so it is fine for iterating on the app
but not for producing a multi-arch image:

```bash
# Python app on http://localhost:8001
DOCKER_IMAGE=your-dockerhub-username/multiarch-lab \
  docker compose -f examples/python-fastapi-multiarch/docker-compose.yml up --build

# C# app on http://localhost:8201
DOCKER_IMAGE=your-dockerhub-username/multiarch-lab \
  docker compose -f examples/csharp-aspnet/docker-compose.yml up --build
```

Try the API:

```bash
curl -s -X POST http://localhost:8001/paste \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://www.docker.com/"}'
# {"code":"Ab12Cd34","short_url":"/Ab12Cd34","long_url":"https://www.docker.com/"}
```

### Building the multi-arch images locally

If you have Docker with Buildx, you can reproduce examples 2 and 3 on your own
machine. The one-time setup registers QEMU binfmt handlers (Docker Desktop does
this for you):

```bash
# One-time: create a buildx builder that can run foreign architectures
docker run --privileged --rm tonistiigi/binfmt --install arm64
docker buildx create --name multiarch --use --bootstrap

# Example 3 equivalent: emulated amd64 + arm64, pushed to Docker Hub
docker login
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t your-dockerhub-username/multiarch-lab:csharp-multiarch-qemu \
  --push examples/csharp-aspnet

# Example 1 equivalent: native ARM64 only
docker buildx build \
  --platform linux/arm64 \
  -t your-dockerhub-username/multiarch-lab:python-arm64 \
  --push examples/python-fastapi-multiarch
```

To build example 2 locally, run the two platform builds separately and combine
the digests exactly like the workflow does:

```bash
docker buildx build --platform linux/amd64 \
  --output type=image,name=your-dockerhub-username/multiarch-lab,push-by-digest=true,name-canonical=true,push=true \
  examples/csharp-aspnet

docker buildx build --platform linux/arm64 \
  --output type=image,name=your-dockerhub-username/multiarch-lab,push-by-digest=true,name-canonical=true,push=true \
  examples/csharp-aspnet
# then combine the two reported digests:
docker buildx imagetools create \
  -t your-dockerhub-username/multiarch-lab:csharp-multiarch-native \
  your-dockerhub-username/multiarch-lab@sha256:<amd64-digest> \
  your-dockerhub-username/multiarch-lab@sha256:<arm64-digest>
```

---

## How each workflow works

### Example 1 - `python-fastapi-multiarch.yml` (native ARM64)

The simplest case. The workflow asks for `runs-on: ubuntu-26.04-arm` and builds
with `platforms: linux/arm64`. Because the runner _is_ ARM64, no emulation is
involved and the build runs at native speed.

```yaml
runs-on: ubuntu-26.04-arm
steps:
  - uses: docker/setup-buildx-action@v4
  - uses: docker/build-push-action@v7
    with:
      context: examples/python-fastapi-multiarch
      platforms: linux/arm64 # native on this runner
      push: true
      tags: ${{ env.DOCKER_IMAGE }}:${{ env.TAG }}
```

### Example 2 - `csharp-aspnet.yml` (native AMD64 + ARM64, then merge)

This is the "each architecture on real hardware" approach, and it is the only
one that needs more than one job.

**Stage 1 - build per architecture.** A matrix runs one job on `ubuntu-26.04`
(amd64) and one on `ubuntu-26.04-arm` (arm64). Each job pushes its image **by
digest** - the image is uploaded but _not tagged_:

```yaml
outputs: type=image,name=${{ env.DOCKER_IMAGE }},push-by-digest=true,name-canonical=true,push=true
```

Both per-architecture builds also set `provenance: false` and `sbom: false`.
Attestations are stored as extra entries in the image index, and
`docker buildx imagetools create` only combines plain image manifests - so they
must be disabled on the per-architecture builds for the merge to succeed.

Each job then writes the returned `sha256:...` digest (`steps.build.outputs.digest`)
to a file and uploads it as an artifact.

**Stage 2 - merge.** A third job downloads all digest artifacts and combines
them into a single manifest list with `docker buildx imagetools create`, then
pushes it under the tag:

```yaml
docker buildx imagetools create \
--tag "${{ env.DOCKER_IMAGE }}:${{ env.TAG }}" \
$(printf '${{ env.DOCKER_IMAGE }}@sha256:%s ' *)
```

The digest artifacts are what let the merge job - which runs on a fresh machine
with no build cache - refer to images that were built elsewhere.

### Example 3 - `pastebin-csharp-qemu.yml` (QEMU on one runner)

One x64 runner does everything. `docker/setup-qemu-action` registers ARM64
`binfmt_misc` handlers so the runner can execute ARM64 binaries through QEMU,
and Buildx builds both platforms in one invocation, pushing the manifest list
directly:

```yaml
- uses: docker/setup-qemu-action@v4
  with:
    platforms: arm64
- uses: docker/setup-buildx-action@v4
- uses: docker/build-push-action@v7
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ${{ env.DOCKER_IMAGE }}:${{ env.TAG }}
```

**Trade-offs at a glance:**

|                      | Example 1 (native, one arch) | Example 2 (native, multi-arch)  | Example 3 (QEMU, multi-arch) |
| -------------------- | ---------------------------- | ------------------------------- | ---------------------------- |
| Runners              | 1 ARM64                      | 1 AMD64 + 1 ARM64 (+1 merge)    | 1 AMD64                      |
| ARM64 build speed    | Fast (native)                | Fast (native)                   | Slow (emulated)              |
| Workflow complexity  | Lowest                       | Highest (matrix + digest merge) | Medium                       |
| Manifest produced by | build-push-action            | `imagetools create`             | build-push-action            |

---

## Runner labels

The workflows use GitHub's Ubuntu 26.04 runners:

| Platform | Label              | Fallback if the label is unavailable |
| -------- | ------------------ | ------------------------------------ |
| x86_64   | `ubuntu-26.04`     | `ubuntu-24.04`                       |
| ARM64    | `ubuntu-26.04-arm` | `ubuntu-24.04-arm`                   |

---

## Troubleshooting

| Symptom                                                           | Cause                                                                                     | Fix                                                                                                |
| ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `denied: requested access to the resource is denied`              | Token missing, wrong, or without write scope; or pushing under a namespace you do not own | Re-check the `DOCKERHUB_TOKEN` secret and that `DOCKER_IMAGE` starts with your Docker Hub username |
| `username or password is incorrect`                               | Token expired or login username derived from `DOCKER_IMAGE` is wrong                      | Regenerate the token; make sure the prefix of `DOCKER_IMAGE` is your exact Docker Hub username     |
| `no runner matching the specified labels`                         | Runner label not available in your account/region                                         | Use the fallback labels in the table above                                                         |
| `unknown flag: --platform` or `buildx: not found`                 | Old Docker on a self-hosted runner                                                        | Ensure `docker/setup-buildx-action` runs before the build step                                     |
| Manifest has only one platform                                    | One matrix job failed, or the merge job ran without all digests                           | Check **Actions → run →** the failed arch job; make sure both `digest-*` artifacts were uploaded   |
| ARM64 image exits with `exec format error` on your laptop         | You tried to run an ARM64 image without QEMU/binfmt                                       | Use Docker Desktop (ships binfmt) or install QEMU as shown above                                   |
| `unexpected status from POST /v2/.../blobs/uploads` while merging | Transient registry error                                                                  | Re-run the failed merge job                                                                        |

---

## Submission

Fork this repository, complete the setup above and run all three workflows.

Then, update the README to include the link to the Docker Hub repository you published to.

Link: <https://hub.docker.com/repository/docker/heniveress/kuberneteslab03/general/>

---

## Further reading

- Multi-platform images - <https://docs.docker.com/build/building/multi-platform/>
- `docker/build-push-action` (incl. the digest-merge recipe) - <https://github.com/docker/build-push-action>
- `docker/setup-qemu-action` - <https://github.com/docker/setup-qemu-action>
- `docker buildx imagetools` - <https://docs.docker.com/reference/cli/docker/buildx/imagetools/>
- GitHub-hosted runners - <https://docs.github.com/actions/using-github-hosted-runners/about-github-hosted-runners>
- OCI image index spec - <https://github.com/opencontainers/image-spec/blob/main/image-index.md>
