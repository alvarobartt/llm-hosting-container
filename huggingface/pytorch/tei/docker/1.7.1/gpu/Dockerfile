FROM nvidia/cuda:12.2.0-devel-ubuntu22.04 AS base-builder

ENV SCCACHE=0.10.0
ENV RUSTC_WRAPPER=/usr/local/bin/sccache
ENV PATH="/root/.cargo/bin:${PATH}"
# aligned with `cargo-chef` version in `lukemathwalker/cargo-chef:latest-rust-1.85-bookworm`
ENV CARGO_CHEF=0.1.71

RUN apt-get update && apt-get upgrade -y && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
    curl \
    libssl-dev \
    pkg-config \
    libgssapi-krb5-2 \
    && rm -rf /var/lib/apt/lists/*

# Donwload and configure sccache
RUN curl -fsSL https://github.com/mozilla/sccache/releases/download/v$SCCACHE/sccache-v$SCCACHE-x86_64-unknown-linux-musl.tar.gz | tar -xzv --strip-components=1 -C /usr/local/bin sccache-v$SCCACHE-x86_64-unknown-linux-musl/sccache && \
    chmod +x /usr/local/bin/sccache

RUN curl https://sh.rustup.rs -sSf | bash -s -- -y
RUN cargo install cargo-chef --version $CARGO_CHEF --locked

FROM base-builder AS planner

WORKDIR /usr/src

COPY backends backends
COPY core core
COPY router router
COPY Cargo.toml ./
COPY Cargo.lock ./

RUN cargo chef prepare  --recipe-path recipe.json

FROM base-builder AS builder

ARG GIT_SHA
ARG DOCKER_LABEL

# sccache specific variables
ARG SCCACHE_GHA_ENABLED

# Limit parallelism
ARG RAYON_NUM_THREADS=4
ARG CARGO_BUILD_JOBS
ARG CARGO_BUILD_INCREMENTAL

WORKDIR /usr/src

COPY --from=planner /usr/src/recipe.json recipe.json

RUN --mount=type=secret,id=actions_results_url,env=ACTIONS_RESULTS_URL \
    --mount=type=secret,id=actions_runtime_token,env=ACTIONS_RUNTIME_TOKEN \
    cargo chef cook --release --recipe-path recipe.json && sccache -s;

RUN --mount=type=secret,id=actions_results_url,env=ACTIONS_RESULTS_URL \
    --mount=type=secret,id=actions_runtime_token,env=ACTIONS_RUNTIME_TOKEN \
    CUDA_COMPUTE_CAP=75 cargo chef cook --release --features candle-cuda-turing --recipe-path recipe.json && sccache -s;

RUN --mount=type=secret,id=actions_results_url,env=ACTIONS_RESULTS_URL \
    --mount=type=secret,id=actions_runtime_token,env=ACTIONS_RUNTIME_TOKEN \
    CUDA_COMPUTE_CAP=80 cargo chef cook --release --features candle-cuda --recipe-path recipe.json && sccache -s;

RUN --mount=type=secret,id=actions_results_url,env=ACTIONS_RESULTS_URL \
    --mount=type=secret,id=actions_runtime_token,env=ACTIONS_RUNTIME_TOKEN \
    CUDA_COMPUTE_CAP=90 cargo chef cook --release --features candle-cuda --recipe-path recipe.json && sccache -s;

COPY backends backends
COPY core core
COPY router router
COPY Cargo.toml ./
COPY Cargo.lock ./

RUN --mount=type=secret,id=actions_results_url,env=ACTIONS_RESULTS_URL \
    --mount=type=secret,id=actions_runtime_token,env=ACTIONS_RUNTIME_TOKEN \
    CUDA_COMPUTE_CAP=75 cargo build --release --bin text-embeddings-router -F candle-cuda-turing && sccache -s;

RUN mv /usr/src/target/release/text-embeddings-router /usr/src/target/release/text-embeddings-router-75

RUN --mount=type=secret,id=actions_results_url,env=ACTIONS_RESULTS_URL \
    --mount=type=secret,id=actions_runtime_token,env=ACTIONS_RUNTIME_TOKEN \
    CUDA_COMPUTE_CAP=80 cargo build --release --bin text-embeddings-router -F candle-cuda && sccache -s;

RUN mv /usr/src/target/release/text-embeddings-router /usr/src/target/release/text-embeddings-router-80

RUN --mount=type=secret,id=actions_results_url,env=ACTIONS_RESULTS_URL \
    --mount=type=secret,id=actions_runtime_token,env=ACTIONS_RUNTIME_TOKEN \
    CUDA_COMPUTE_CAP=90 cargo build --release --bin text-embeddings-router -F candle-cuda && sccache -s;

RUN mv /usr/src/target/release/text-embeddings-router /usr/src/target/release/text-embeddings-router-90

FROM nvidia/cuda:12.2.0-runtime-ubuntu22.04 AS base

ARG DEFAULT_USE_FLASH_ATTENTION=True

ENV HUGGINGFACE_HUB_CACHE=/opt/ml/model \
    PORT=80 \
    USE_FLASH_ATTENTION=$DEFAULT_USE_FLASH_ATTENTION

RUN apt-get update && apt-get upgrade -y && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
    ca-certificates \
    libssl-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/src/target/release/text-embeddings-router-75 /usr/local/bin/text-embeddings-router-75
COPY --from=builder /usr/src/target/release/text-embeddings-router-80 /usr/local/bin/text-embeddings-router-80
COPY --from=builder /usr/src/target/release/text-embeddings-router-90 /usr/local/bin/text-embeddings-router-90

# Amazon SageMaker compatible image
FROM base AS sagemaker

ENV HF_HUB_USER_AGENT_ORIGIN=aws:sagemaker:gpu-cuda:inference:tei

COPY --chmod=775 sagemaker-entrypoint-cuda-all.sh entrypoint.sh
COPY --chmod=775 /huggingface/pytorch/tei/docker/1.7.1/gpu/start-cuda-compat.sh start-cuda-compat.sh

ENTRYPOINT ["./entrypoint.sh"]
