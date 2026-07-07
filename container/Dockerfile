# =========================================================
# Stage 1: base
# =========================================================
FROM ubuntu:24.04 AS base

WORKDIR /opt
COPY packages packages

RUN apt-get update \
   && apt-get install -y $(cat packages) \
   && rm -rf /var/lib/apt/lists/*

# =========================================================
# Stage 2: prebuild
#
# Stable, checksum-pinned downloads only (ROOT, buuinput). This stage
# never references GiBUU source, so it stays a Docker cache hit across
# every rebuild that only touches GiBUU/RootTuple code -- only rerun
# when packages/checksums/URLs in fetch-stable itself change.
# =========================================================
FROM base AS prebuild

WORKDIR /opt
COPY fetch-stable fetch-stable

RUN ./fetch-stable download root
RUN ./fetch-stable download buuinput
RUN ./fetch-stable checksum
RUN ./fetch-stable unpack

# =========================================================
# Stage 3: builder
#
# Volatile GiBUU/RootTuple source checkout + compile. GIBUU_REF and
# ROOTTUPLE_REF are build args so that pointing them at a new commit
# (e.g. --build-arg GIBUU_REF=$(git ls-remote ... main)) changes this
# layer's cache key and forces a rebuild -- without touching the
# prebuild stage above.
# =========================================================
FROM prebuild AS builder

WORKDIR /opt
COPY gibuu-builder gibuu-builder

ARG GIBUU_REF=main
ARG ROOTTUPLE_REF=main

RUN ./gibuu-builder download gibuu "${GIBUU_REF}"
RUN ./gibuu-builder download roottuple "${ROOTTUPLE_REF}"
RUN ./gibuu-builder version
RUN ./gibuu-builder build
RUN ./gibuu-builder install /opt/bin

# =========================================================
# Stage 4: runtime
# =========================================================
FROM base AS runtime

WORKDIR /opt

COPY --from=builder /opt/root /opt/root
COPY --from=builder /opt/gibuu /opt/gibuu
COPY --from=builder /opt/bin /opt/bin

ENV ROOTSYS=/opt/root
ENV PATH="${ROOTSYS}/bin:/opt/bin:${PATH}"
ENV LD_LIBRARY_PATH="${ROOTSYS}/lib"

CMD ["GiBUU.x"]
