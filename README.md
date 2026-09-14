# 302 Docker Neuroimaging

A containerized neuroimaging workflow that bundles AFNI, a Python environment, and a notebook-based analysis pipeline for MRI processing.

## Overview

This project packages the tools needed for a reproducible MRI analysis workflow inside a Docker image. It is built around AFNI and a Jupyter environment so the pipeline can be run consistently across machines.

Three Dockerfiles are provided:

| File                    | Target platform    | ANTs | Notes                                                         |
| ----------------------- | ------------------ | ---- | ------------------------------------------------------------- |
| `Dockerfile`            | `linux/amd64` only | yes  | The reference build. Emulated on Apple Silicon and ARM Linux. |
| `Dockerfile.macos`      | Host architecture  | no   | Native arm64/amd64. Builds in minutes. Start here.            |
| `Dockerfile.macos-ants` | Host architecture  | yes  | Native, but compiles ANTs from source. Slow build, needs RAM. |

Despite the name, the `.macos` files are not macOS-specific — they build for whatever architecture the Docker host is, so they are also the right choice on ARM Linux. See [Choosing a Dockerfile](#choosing-a-dockerfile).

**Use `Dockerfile.macos` unless you specifically need ANTs.** The bundled notebook does not use ANTs, and `Dockerfile.macos-ants` trades a multi-minute build for one that can run well over an hour.

## Requirements

- Docker (Docker Desktop on macOS, Docker Engine on Linux)
- Roughly 10 GB of free disk space for the build

For `Dockerfile` or `Dockerfile.macos`:

- 8 GB RAM on the host is comfortable

For `Dockerfile.macos-ants`, which compiles ANTs from source:

- **At least 8 GB allocated to the Docker VM**, not just present on the host. This is the number that matters, and it is not the same thing — Docker Desktop hands the VM a fraction of host RAM by default.
- Roughly 25 GB of free disk for the build tree
- On a base-model 8 GB M1 this variant is **not practical**: the VM typically gets 2–4 GB, the build drops to one or two parallel jobs, and it can take many hours. Use `Dockerfile.macos` instead, or raise the VM's memory allocation first.

## Clone the repository

```bash
git clone https://github.com/oesteban/302-docker-neuroimaging.git
cd 302-docker-neuroimaging
```

## Build and run

Pick the section matching your machine. In every case, the container prints a URL with an access token when it starts:

```text
http://127.0.0.1:8888/tree?token=<token>
```

Open that whole URL — including the token — in your browser, then use the notebook named `brain_mri_pipeline.ipynb`.

### macOS — Apple Silicon (M1/M2/M3/M4)

```bash
docker build -f Dockerfile.macos -t neuropipeline-302 .
docker run -it --rm -p 8888:8888 neuropipeline-302
```

No `--platform` flag is needed: you get native `arm64` AFNI binaries, with no Rosetta or QEMU emulation.

### macOS — Intel

```bash
docker build -f Dockerfile.macos -t neuropipeline-302 .
docker run -it --rm -p 8888:8888 neuropipeline-302
```

Same commands as Apple Silicon — the build selects the `amd64` AFNI package automatically.

### Linux — x86_64

The reference `Dockerfile` builds natively here, and ships ANTs prebuilt:

```bash
docker build -t neuropipeline-302 .
docker run -it --rm -p 8888:8888 neuropipeline-302
```

`Dockerfile.macos` also builds natively on x86_64 Linux if you prefer the slimmer, conda-free image:

```bash
docker build -f Dockerfile.macos -t neuropipeline-302 .
```

### Linux — arm64 / aarch64

Use `Dockerfile.macos`; the reference `Dockerfile` is amd64-only and would need emulation:

```bash
docker build -f Dockerfile.macos -t neuropipeline-302 .
docker run -it --rm -p 8888:8888 neuropipeline-302
```

### Any platform — with ANTs

Only if you actually need ANTs. Read [Requirements](#requirements) first; on a small Docker VM this build is impractical.

```bash
docker build -f Dockerfile.macos-ants -t neuropipeline-302:ants .
docker run -it --rm -p 8888:8888 neuropipeline-302:ants
```

See [ANTs on arm64](#ants-on-arm64) for why this has to be compiled rather than installed.

### Linux — running as a non-1000 user

The container runs as `databot`, pinned to UID/GID 1000 to match the reference image. Linux does not remap bind-mount ownership the way Docker Desktop does, so if your host account is not UID 1000, build with your own IDs to keep mounted output directories writable:

```bash
docker build -f Dockerfile.macos \
  --build-arg DATABOT_UID="$(id -u)" \
  --build-arg DATABOT_GID="$(id -g)" \
  -t neuropipeline-302 .
```

Check yours with `id -u`. If it already prints `1000`, the plain build is fine.

### Rootless Docker / Podman on Linux

Under rootless Docker or Podman, user namespaces already map the container user onto your host account, so no `--build-arg` is needed. With Podman, add `:Z` to bind mounts on SELinux systems (Fedora, RHEL):

```bash
podman run -it --rm -p 8888:8888 -v "$PWD/outputs:/home/databot/outputs:Z" neuropipeline-302
```

## Useful commands

### Rebuild after changes

```bash
docker build -f Dockerfile.macos      -t neuropipeline-302 .        # native, no ANTs
docker build -f Dockerfile.macos-ants -t neuropipeline-302:ants .   # native, with ANTs
docker build                          -t neuropipeline-302 .        # reference, amd64
```

Layers are cached, so an edit to the notebook or the README rebuilds only the last few steps. The one exception is the ANTs compile, which is its own stage and is not re-run unless `ANTS_VERSION` or the base image changes.

### Run the container interactively

```bash
docker run -it --rm -p 8888:8888 neuropipeline-302 /bin/bash
```

### Persist your outputs on the host

```bash
mkdir -p outputs
docker run -it --rm -p 8888:8888 -v "$PWD/outputs:/home/databot/outputs" neuropipeline-302
```

On macOS the directory must live under a path Docker Desktop is allowed to share — your home directory and the cloned repository qualify by default. If you bind-mount a directory outside the shared list, Docker silently substitutes an empty root-owned directory inside its VM and the container cannot write to it. Check Docker Desktop → Settings → Resources → File sharing.

On Linux, see [Linux — running as a non-1000 user](#linux--running-as-a-non-1000-user) if the container cannot write to the mount.

### Stop a running container

```bash
docker ps
# then stop the specific container with:
docker stop <container_id>
```

## Choosing a Dockerfile

`Dockerfile` is pinned to `linux/amd64` for good reasons — it uses AFNI's static `linux_openmp_64` binaries together with back-ported `libpng12`, `libxp6` and `multiarch-support` packages, and its `nipreps/miniconda` base image is published for amd64 only. None of that has an arm64 equivalent.

`Dockerfile.macos` reaches the same result along a different path:

- **Base image:** `ubuntu:noble`, which is published for both `arm64` and `amd64`.
- **AFNI:** the Ubuntu 24.04 *shared* builds, which NIMH publishes for both ARM64 (`linux_ubuntu_24_ARM.tgz`) and x86_64 (`linux_ubuntu_24_64.tgz`). The correct one is selected automatically by asking the build container itself, via `dpkg --print-architecture`. These link against stock Noble libraries, so the libpng12/libxp6/multiarch-support back-porting disappears entirely.
- **Python tooling:** a plain `venv` with `datalad`, `ipyniivue` and `jupyter` from PyPI, instead of conda/micromamba. Every wheel involved is pure Python or ships arm64 builds.
- **`git-annex`:** installed from Ubuntu rather than conda-forge, which publishes it for `linux-64` only.

Because the Ubuntu 24 builds are *shared* rather than static, they ship their own `libmri.so` and `libSUMA.so`. The native image therefore keeps the full AFNI distribution instead of pruning it down to a handful of binaries the way `Dockerfile` does, which makes the image somewhat larger (~490 MB).

## ANTs on arm64

`Dockerfile.macos` does **not** install ANTs, because nobody publishes a prebuilt ANTs for `linux/arm64`:

| Source                       | linux/arm64 build?                                 |
| ---------------------------- | -------------------------------------------------- |
| conda-forge                  | No — `linux-64`, `osx-64`, `osx-arm64` only        |
| Ubuntu noble apt             | No — `ants` has no installation candidate at all   |
| ANTsX/ANTs releases (v2.6.5) | No — every Linux asset is X64                      |
| PyPI `antspyx`               | No — Linux wheels are `manylinux_2_17_x86_64` only |
| Docker Hub `antsx/ants`      | No — all tags are `linux/amd64`                    |

The ARM64 builds that do exist (`macos-14-ARM64`, `osx-arm64`, the macOS `antspyx` wheels) are *Darwin* binaries and cannot run in a Linux container.

This does not affect the bundled notebook, which only uses AFNI (`3dSkullStrip`, `3dSeg`, `3dAFNItoNIFTI`, `3dAutomask`).

### Getting ANTs anyway

Two options.

**Compile it natively** with `Dockerfile.macos-ants`. ANTs is C++/ITK with no x86 intrinsics, so it builds cleanly on aarch64 — it is just slow:

```bash
docker build -f Dockerfile.macos-ants -t neuropipeline-302:ants .
```

The `ants-builder` stage runs a CMake superbuild: ITK first (~2,500 sources), then ~200 heavily-templated ANTs programs. Read [Requirements](#requirements) before starting — memory, not cores, is usually the limit.

Job count is chosen automatically as `min(nproc, 2/3 × RAM_GB)` rather than plain `nproc`, because each linker process can peak near 2 GB and a naive `-j8` on a small VM will OOM. Override it if you know better:

```bash
docker build -f Dockerfile.macos-ants --build-arg ANTS_BUILD_JOBS=4 -t neuropipeline-302:ants .
```

Pin a different ANTs release with `--build-arg ANTS_VERSION=v2.6.4`. Because ANTs is a separate build stage, it is cached: editing the notebook or the Python layer does not recompile it. A `docker system prune` does discard it, and you pay the full build again.

**Or emulate amd64** and use the reference image, which ships ANTs prebuilt:

```bash
docker buildx build --platform linux/amd64 --load -t neuropipeline-302-amd64 .
docker run --platform linux/amd64 -it --rm -p 8888:8888 neuropipeline-302-amd64
```

This needs amd64 emulation registered in your Docker engine. If `docker buildx inspect` lists only `linux/arm64` under Platforms, there is no emulator and the build will crawl or stall rather than fail cleanly — install one with `docker run --privileged tonistiigi/binfmt --install amd64`. Note that Rosetta-backed emulation is a Docker Desktop feature; other engines fall back to QEMU, which is considerably slower.

## Included tools

- AFNI (native to the build architecture)
- Jupyter Notebook / JupyterLab-compatible environment
- `ipyniivue` for interactive 3D volume viewing
- `datalad` + `git-annex`, with the `ds000005` dataset preloaded under `~/data`
- ANTs — reference `Dockerfile` and `Dockerfile.macos-ants` only

## License

This project is distributed under the MIT license. See the LICENSE file for details.
