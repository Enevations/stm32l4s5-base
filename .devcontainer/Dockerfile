FROM debian:trixie

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    ca-certificates \
    cmake \
    gcc-arm-none-eabi \
    gdb-multiarch \
    binutils-arm-none-eabi \
    libnewlib-arm-none-eabi \
    libstdc++-arm-none-eabi-newlib \
    ninja-build \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace