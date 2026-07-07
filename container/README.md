# GiBUU Data Production Container

Multi-stage container build that produces a runtime image containing
`GiBUU.x` + ROOT, for running the GiBUU cascade simulation in data
production.

## Layout

- `Dockerfile` — 4 stages: `base` → `prebuild` → `builder` → `runtime`.
- `packages` — apt package list installed in `base`.
- `fetch-stable` — downloads, checksums, and unpacks ROOT + buuinput.
  Used only in `prebuild`.
- `gibuu-builder` — checks out GiBUU + RootTuple source, builds, and
  installs `GiBUU.x`. Used only in `builder`.
- `.dockerignore` — keeps `.git`/`notes` out of the build context.

## Why two scripts, two stages

ROOT and buuinput are large, checksum-pinned, and change rarely.
GiBUU/RootTuple source is a git checkout that changes on every push.
Keeping their download logic in separate files, copied into separate
stages, means editing GiBUU build/install logic never invalidates the
Docker cache for the slow ROOT/buuinput downloads, and vice versa:

- `prebuild` runs `fetch-stable` — cache hit on every build unless
  `packages`, `fetch-stable` itself, or the pinned URLs/checksums
  change.
- `builder` runs `gibuu-builder` — re-clones GiBUU/RootTuple and
  rebuilds whenever `GIBUU_REF`/`ROOTTUPLE_REF` change (see below).

## Building

Run from this directory (`src/gibuu-dataprod/container/`).

### Docker

```sh
docker build -t gibuu:local .
```

### Podman

```sh
podman build -t gibuu:local .
```

Both default `GIBUU_REF` and `ROOTTUPLE_REF` to `main`. To build a
specific tag/branch of either repo instead:

```sh
docker build \
  --build-arg GIBUU_REF=my-feature-branch \
  --build-arg ROOTTUPLE_REF=main \
  -t gibuu:local .
```

`GIBUU_REF`/`ROOTTUPLE_REF` must be a **branch or tag name**, not a
raw commit SHA — the underlying `git clone --branch --depth 1` does
not accept arbitrary SHAs against GitHub.

### Debugging a single stage

```sh
docker build --target prebuild -t gibuu:prebuild .
docker build --target builder  -t gibuu:builder .
```

(same flags work with `podman build`)

## Rebuilding after a GiBUU/RootTuple source change

Passing `--build-arg GIBUU_REF=main` again does **not** by itself
force a re-clone: the build-arg value is unchanged, so the cache key
for that layer is unchanged, so both Docker and Podman will happily
reuse the stale clone from a previous build. Pick one:

- **Pin to a tag/branch you actually cut** for the new state (e.g. a
  release tag) and pass it as `GIBUU_REF`/`ROOTTUPLE_REF`. Preferred —
  reproducible, and works identically on Docker and Podman.
- **Docker/buildx only** — force just the volatile stage to rerun
  without touching the cached ROOT/buuinput layers:
  ```sh
  docker buildx build --no-cache-filter builder -t gibuu:local .
  ```
- **Podman** has no per-stage cache flag. The portable fallback is a
  full `podman build --no-cache -t gibuu:local .`, which also
  re-downloads ROOT/buuinput — slow, use only when needed.

## Running

```sh
docker run --rm -i gibuu:local < path/to/jobcard.job
```

`CMD` runs `GiBUU.x`, which reads its job card from stdin (see
upstream docs: https://gibuu.hepforge.org/trac/wiki/running). Mount a
working directory for job cards/output as needed, e.g. `-v
"$PWD":/opt/run -w /opt/run`.

`ROOTSYS`, `PATH`, and `LD_LIBRARY_PATH` are already set in the image;
no extra environment setup is required.

## Troubleshooting

- **`Checksum verification failed`** — the downloaded ROOT or buuinput
  tarball doesn't match the pinned md5sum in `fetch-stable`. Either
  the upstream file changed or the download was corrupted; delete any
  cached `prebuild` image/layer and retry before touching the
  checksums.
- **`Unknown package: <name>`** — typo in a `download`/`checksum`
  argument; valid names are `root`/`buuinput` for `fetch-stable` and
  `gibuu`/`roottuple` for `gibuu-builder`.
