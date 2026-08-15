# ============================================================================
# GraphRAG Server — Multi-stage Dockerfile
# ============================================================================
# Build:   docker build -t graphrag-server -f graphrag-server/Dockerfile .
# Run:     docker run -p 8080:8080 graphrag-server
# ============================================================================

# --- Stage 1: Build ---
FROM rust:1.82-slim-bookworm AS builder

RUN apt-get update && apt-get install -y pkg-config libssl-dev && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY . .

RUN cargo build --release -p graphrag-server

# --- Stage 2: Runtime ---
FROM debian:bookworm-slim

# No apt-get here: this build environment can't reach deb.debian.org (every
# mirror gets Ign:/Err: on `apt-get update`, reproduced twice, deterministic).
# ca-certificates and libssl3 are copied straight from the builder image
# instead, which already has them as part of its own base layers.
COPY --from=builder /etc/ssl/certs /etc/ssl/certs
COPY --from=builder /usr/lib/x86_64-linux-gnu/libssl.so* /usr/lib/x86_64-linux-gnu/
COPY --from=builder /usr/lib/x86_64-linux-gnu/libcrypto.so* /usr/lib/x86_64-linux-gnu/

COPY --from=builder /app/target/release/graphrag-server /usr/local/bin/graphrag-server

# Non-root user
RUN useradd -r -s /bin/false graphrag
USER graphrag

# Default environment variables
ENV EMBEDDING_BACKEND=hash \
    EMBEDDING_DIM=384 \
    JWT_SECRET=change-me-in-production \
    QDRANT_URL=http://qdrant:6334 \
    COLLECTION_NAME=graphrag

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD ["/usr/local/bin/graphrag-server", "--version"] || exit 1

ENTRYPOINT ["graphrag-server"]
