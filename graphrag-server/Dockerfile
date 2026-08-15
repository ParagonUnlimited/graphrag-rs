# ============================================================================
# GraphRAG Server — Multi-stage Dockerfile
# ============================================================================
# Build:   docker build -t graphrag-server -f graphrag-server/Dockerfile .
# Run:     docker run -p 8080:8080 graphrag-server
# ============================================================================

# --- Stage 1: Build ---
FROM rust:1.82-slim-bookworm AS builder

# No apt-get here: this build environment cannot reach deb.debian.org at all
# (every mirror returns "Connection failed" to its Fastly-hosted IP,
# reproduced deterministically in both stages). pkg-config/libssl-dev are no
# longer needed - graphrag-server/Cargo.toml pins openssl's "vendored"
# feature, which compiles OpenSSL from its own bundled source instead of
# linking a system libssl, so the whole workspace builds with zero apt calls.
WORKDIR /app
COPY . .

# --features ollama: compiles in the real Ollama-backed embedding client
# (ollama-rs) alongside the default "qdrant" feature, instead of only the
# hash fallback.
RUN cargo build --release -p graphrag-server --features ollama

# --- Stage 2: Runtime ---
FROM debian:bookworm-slim

# No apt-get here either, for the same reason. ca-certificates isn't needed:
# graphrag-server only calls http:// (Qdrant, Ollama) at runtime, never TLS.
# OpenSSL is statically linked in via the vendored build, so there's no
# libssl.so runtime dependency to satisfy either.
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
