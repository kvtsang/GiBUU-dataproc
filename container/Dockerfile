FROM ubuntu:24.04

WORKDIR /opt

COPY packages packages
COPY gibuu_helper gibuu_helper

RUN apt update -y
RUN apt install -y --no-install-recommends $(cat packages)
RUN apt-get autoremove -y && apt-get clean -y \
  && rm -rf /var/lib/apt/lists/* \
  && rm -f packages

RUN ./gibuu_helper download && ./gibuu_helper checksum 
RUN ./gibuu_helper unpack 
RUN ./gibuu_helper build 
RUN ./gibuu_helper install /opt/bin 
RUN ./gibuu_helper clean; rm -f gibuu_helper

ENV ROOTSYS=/opt/root
ENV PATH="${ROOTSYS}/bin:/opt/bin:${PATH}"
ENV LD_LIBRARY_PATH="${ROOTSYS}/lib"

#CMD ["GiBUU.x"]
