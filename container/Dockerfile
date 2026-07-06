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
# =========================================================
FROM base AS prebuild

WORKDIR /opt
COPY gibuu-builder gibuu-builder
RUN ./gibuu-builder download 

# =========================================================
# Stage 3: builder 
# =========================================================
FROM prebuild AS builder

WORKDIR /opt

# fIXME
RUN ./gibuu-builder checksum; exit 0
RUN ./gibuu-builder unpack
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
