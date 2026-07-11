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
docker build \
  --build-arg GIBUU_REF=v1.2.3 \
  --build-arg ROOTTUPLE_REF=v1.2.3 \
  -t gibuu:local .
```

### Podman

```sh
podman build \
  --build-arg GIBUU_REF=v1.2.3 \
  --build-arg ROOTTUPLE_REF=v1.2.3 \
  -t gibuu:local .
```

`GIBUU_REF` and `ROOTTUPLE_REF` have **no default** — a bare
`docker build .`/`podman build .` fails fast in the `builder` stage
with a missing-build-arg error rather than silently building against
the moving `main` branch. Both are required on every invocation.

`GIBUU_REF`/`ROOTTUPLE_REF` accept a tag or a commit SHA (7-40 hex
chars). A full 40-char SHA is fetched shallow (`git fetch --depth 1`,
which GitHub serves for a full SHA); an abbreviated SHA falls back to
a full clone since GitHub won't resolve a short SHA server-side.

A branch name (including `main`, `master`, `develop`) is refused with
a "Refusing to build against moving branch" error — a branch keeps
moving underneath a multi-week production campaign, unlike a tag or
SHA. For local testing against a branch, opt in explicitly:

```sh
docker build \
  --build-arg GIBUU_REF=my-feature-branch \
  --build-arg ROOTTUPLE_REF=v1.2.3 \
  --build-arg ALLOW_BRANCH_REF=1 \
  -t gibuu:local .
```

Never pass `ALLOW_BRANCH_REF` for a build feeding a production
campaign.

The resolved refs are stamped onto the runtime image as
`gibuu_ref`/`roottuple_ref` OCI labels, so `docker inspect
--format '{{json .Config.Labels}}' gibuu:local` (or `podman inspect`)
recovers exactly what was built even without re-reading the build
log.

### Debugging a single stage

```sh
docker build --target prebuild -t gibuu:prebuild .
docker build --target builder --build-arg GIBUU_REF=v1.2.3 \
  --build-arg ROOTTUPLE_REF=v1.2.3 -t gibuu:builder .
```

(same flags work with `podman build`)

## Rebuilding after a GiBUU/RootTuple source change

Passing the same `--build-arg GIBUU_REF=<ref>` again does **not** by
itself force a re-clone: the build-arg value is unchanged, so the
cache key for that layer is unchanged, so both Docker and Podman will
happily reuse the stale clone from a previous build. Pick one:

- **Pin to the new tag/commit SHA you actually cut** for the new
  state (a release tag, or the exact commit SHA) and pass it as
  `GIBUU_REF`/`ROOTTUPLE_REF`. Preferred — reproducible, and works
  identically on Docker and Podman.
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
- **`GIBUU_REF and ROOTTUPLE_REF must be set explicitly`** — no
  `--build-arg` was passed for one or both; see Building above.
- **`Refusing to build against moving branch`** — `GIBUU_REF`/
  `ROOTTUPLE_REF` resolved to `main`/`master`/`develop`. Pin to a tag
  or commit SHA, or pass `--build-arg ALLOW_BRANCH_REF=1` for local
  testing only.
