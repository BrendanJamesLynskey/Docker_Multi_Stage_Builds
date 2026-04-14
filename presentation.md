# Docker Multi-Stage Builds

**Smaller images, faster deployments, cleaner pipelines**

Tags: `Multi-Stage` `BuildKit` `Optimization` `Images`

---

## 01 — Why Image Size Matters

### Security
- Fewer packages = smaller attack surface
- No compilers or dev tools in production
- Fewer CVEs to patch

### Performance
- Faster image pulls & pushes
- Quicker container start-up
- Lower registry storage costs

### Operations
- Faster CI/CD pipelines
- Less bandwidth on deploy
- Easier to audit and scan

A typical Node.js app image can drop from **1.2 GB** to **80 MB** with multi-stage builds and the right base image.

---

## 02 — Single-Stage vs Multi-Stage

### Single-Stage (the problem)

```dockerfile
# Everything in one image
FROM node:20
WORKDIR /app
COPY . .
RUN npm ci && npm run build
EXPOSE 3000
CMD ["node", "dist/server.js"]
# Result: ~1.1 GB image with build tools, node_modules, src...
```

### Multi-Stage (the solution)

```dockerfile
# Stage 1: build
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: production
FROM node:20-slim
WORKDIR /app
COPY --from=build /app/dist ./dist
CMD ["node", "dist/server.js"]
# Result: ~180 MB image, production only
```

Multi-stage builds let you use **multiple FROM statements** — only the final stage becomes the shipped image.

---

## 03 — FROM ... AS Syntax

Each `FROM` instruction starts a **new build stage**. Name stages with `AS` for clarity.

```dockerfile
FROM golang:1.22 AS builder        # Stage 0 - named "builder"
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app ./cmd/server

FROM alpine:3.19 AS certs          # Stage 1 - named "certs"
RUN apk add --no-cache ca-certificates

FROM scratch AS runtime             # Stage 2 - named "runtime" (final)
COPY --from=certs /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

- **Named Stages** — Use descriptive names: `builder`, `deps`, `runtime`
- **Numbered Stages** — `COPY --from=0` works but is fragile
- **External Images** — `COPY --from=nginx:alpine` pulls from any image

---

## 04 — COPY --from Between Stages

`COPY --from` is the bridge between stages — selectively copy only what you need.

### From a Named Stage

```dockerfile
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json .
COPY --from=deps /app/node_modules ./node_modules
```

### From an External Image

```dockerfile
COPY --from=busybox:uclibc /bin/sh /bin/sh
COPY --from=redis:7 /usr/local/bin/redis-cli /usr/local/bin/
```

### Key Rules

- Only **files & directories** are copied — not ENV, USER, or WORKDIR settings
- Ownership resets to `root:root` — use `COPY --from=build --chown=app:app` to fix
- Symlinks are followed and resolved during copy

---

## 05 — Building for Different Environments

Use `--target` to stop at a specific stage — one Dockerfile, multiple outputs.

```dockerfile
FROM node:20 AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM base AS dev                   # docker build --target dev
COPY . .
CMD ["npm", "run", "dev"]          # Hot reload, debug tools

FROM base AS test                  # docker build --target test
COPY . .
RUN npm run lint && npm test

FROM base AS build
COPY . .
RUN npm run build

FROM node:20-slim AS prod          # docker build --target prod (default)
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER node
CMD ["node", "dist/server.js"]
```

Flow: `base` → `dev` | `test` | `build` → `prod`

---

## 06 — Language Pattern: Go

Go compiles to a **static binary** — the ideal multi-stage candidate. Final image can be `scratch`.

```dockerfile
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache git ca-certificates
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w" -o /bin/server ./cmd/server

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /bin/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

| Stage | Size |
|-------|------|
| Build Stage | ~800 MB (Go SDK + deps) |
| Final Image | ~8 MB (static binary + certs) |
| Savings | ~99% size reduction |

---

## 07 — Language Patterns: Java & Python

### Java (Maven + JRE)

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine
COPY --from=build /app/target/*.jar /app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
# JDK build: ~700MB -> JRE runtime: ~190MB
```

### Python (pip + slim)

```dockerfile
FROM python:3.12 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
# Full Python: ~1GB -> Slim: ~150MB
```

**Tip:** For Java, use `jlink` to create a custom JRE and shrink the runtime further.

---

## 08 — Language Patterns: Node.js & Rust

### Node.js (production deps only)

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
CMD ["dist/server.js"]
# node:20 ~1.1GB -> distroless ~130MB
```

### Rust (static musl binary)

```dockerfile
FROM rust:1.77 AS builder
RUN rustup target add x86_64-unknown-linux-musl
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs
RUN cargo build --release --target x86_64-unknown-linux-musl
COPY src ./src
RUN touch src/main.rs && \
    cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/app /
ENTRYPOINT ["/app"]
# rust:1.77 ~1.5GB -> scratch ~5MB
```

---

## 09 — Distroless & Scratch Images

The final base image determines your **security posture** and **image size**.

| Base Image | Size | Shell? | Package Mgr? | Best For |
|------------|------|--------|--------------|----------|
| `ubuntu:24.04` | ~78 MB | Yes | apt | General purpose, debugging |
| `alpine:3.19` | ~7 MB | Yes | apk | Small images with shell access |
| `distroless/static` | ~2 MB | No | No | Go, Rust, static binaries |
| `distroless/cc` | ~5 MB | No | No | C/C++ apps needing libc |
| `distroless/nodejs20` | ~130 MB | No | No | Node.js production |
| `scratch` | **0 bytes** | No | No | Truly static binaries only |

**Distroless Advantage:** No shell means attackers cannot `exec` into the container. No package manager means no install-time exploits. Use `debug` variants (e.g., `distroless/static:debug`) only in development.

---

## 10 — Build Cache Optimization

Docker caches each layer. **Order matters** — put things that change least at the top.

### Cache-Busting Order (Bad)

```dockerfile
FROM node:20
WORKDIR /app
COPY . .                # Any change busts cache
RUN npm ci              # Always re-runs
RUN npm run build       # Always re-runs
```

### Cache-Friendly Order (Good)

```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./   # Changes rarely
RUN npm ci              # Cached if lock unchanged
COPY . .                # Only source changes
RUN npm run build       # Rebuilds only app
```

### Layer Ordering Best Practices

1. System packages and dependencies (change rarely)
2. Language dependency files (`package.json`, `go.mod`, `requirements.txt`)
3. Install dependencies (`npm ci`, `go mod download`, `pip install`)
4. Application source code (changes often)
5. Build commands (`npm run build`, `go build`)

---

## 11 — ARG & Conditional Stages

Use `ARG` to parameterize builds and select stages dynamically.

```dockerfile
ARG GO_VERSION=1.22
ARG TARGET_ENV=prod

FROM golang:${GO_VERSION} AS builder
ARG TARGET_ENV
WORKDIR /src
COPY . .
RUN if [ "$TARGET_ENV" = "dev" ]; then \
      go build -gcflags="all=-N -l" -o /app .; \
    else \
      go build -ldflags="-s -w" -o /app .; \
    fi

FROM alpine:3.19 AS dev
COPY --from=builder /app /app
RUN apk add --no-cache curl jq  # debug tools
CMD ["/app"]

FROM scratch AS prod
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

```bash
# Build-time selection
docker build --build-arg TARGET_ENV=dev --target dev -t myapp:dev .

# Version pinning
docker build --build-arg GO_VERSION=1.21 -t myapp:go1.21 .
```

---

## 12 — BuildKit Features

Enable with `DOCKER_BUILDKIT=1` or use `docker buildx`. BuildKit unlocks powerful caching and secret handling.

### Cache Mounts

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /app .

# Python pip cache
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

### Build Secrets

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20 AS build
WORKDIR /app
COPY . .
RUN --mount=type=secret,id=npmrc,target=/app/.npmrc \
    npm ci

# Build command:
# docker build --secret id=npmrc,src=.npmrc .
```

### More BuildKit Powers

- **SSH forwarding:** `RUN --mount=type=ssh git clone ...` — forward agent without copying keys
- **Bind mounts:** `RUN --mount=type=bind,from=stage,source=/data,target=/mnt` — read without copy
- **Parallel builds:** Independent stages build concurrently — massive speed gains
- **Inline cache:** `--build-arg BUILDKIT_INLINE_CACHE=1` embeds cache metadata in the image

---

## 13 — Image Size Comparison

Real-world impact of multi-stage builds across languages:

| Language | Single-Stage | Multi-Stage | + Distroless/Scratch | Reduction |
|----------|-------------|-------------|---------------------|-----------|
| Go (REST API) | 850 MB | 15 MB (alpine) | **8 MB** (scratch) | **99%** |
| Node.js (Express) | 1.1 GB | 250 MB (slim) | **130 MB** (distroless) | **88%** |
| Java (Spring Boot) | 700 MB | 300 MB (JRE) | **190 MB** (JRE-alpine) | **73%** |
| Python (Flask) | 1.0 GB | 180 MB (slim) | **150 MB** (slim+cleanup) | **85%** |
| Rust (Actix) | 1.5 GB | 12 MB (alpine) | **5 MB** (scratch) | **99.7%** |

Compiled languages (Go, Rust) see the biggest gains because the entire toolchain is excluded. Interpreted languages (Python, Node.js) still benefit significantly by removing dev dependencies and build artifacts.

---

## 14 — .dockerignore Best Practices

A good `.dockerignore` is the **first line of defence** — it shrinks your build context before any stage runs.

### Comprehensive .dockerignore

```
# Version control
.git
.gitignore

# Dependencies (rebuilt in image)
node_modules
vendor
__pycache__
*.pyc

# Build outputs (rebuilt in image)
dist
build
target

# IDE & editor
.vscode
.idea
*.swp

# Environment & secrets
.env
.env.*
*.pem
*.key

# Docker files (avoid recursion)
Dockerfile*
docker-compose*
.dockerignore

# Documentation & CI
README.md
docs/
.github/
.gitlab-ci.yml
```

### Why It Matters

- **Speed:** Less data sent to Docker daemon
- **Security:** Secrets never enter the build context
- **Caching:** Irrelevant file changes do not bust cache
- **Size:** `.git` alone can be hundreds of MB

### Pro Tips

- Use `!` for exceptions: `!src/` allows only src
- Place more specific rules *after* general ones
- Test with `docker build --no-cache` to verify
- BuildKit supports `.dockerignore` per Dockerfile: `Dockerfile.dev.dockerignore`

---

## 15 — CI/CD Integration with Multi-Stage

Multi-stage builds simplify CI pipelines — the Dockerfile *is* the build script.

### GitHub Actions Example

```yaml
name: Build & Push
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v5
        with:
          context: .
          target: prod
          push: true
          tags: ghcr.io/org/app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Multi-Target Pipeline

```yaml
jobs:
  test:
    steps:
      - run: |
          docker build --target test -t myapp:test .
          docker run myapp:test

  deploy:
    needs: test
    steps:
      - run: |
          docker build --target prod -t myapp:prod .
          docker push myapp:prod
```

### CI Cache Strategies

- **GitHub Actions:** `cache-from: type=gha` uses GHA cache backend
- **Registry cache:** `cache-from: type=registry,ref=myapp:cache`
- **Local cache:** `cache-from: type=local,src=/tmp/.buildx-cache`

---

## 16 — Debugging Intermediate Stages

When builds fail, you need to inspect what happened **inside** a stage.

### Build & Stop at a Stage

```bash
# Stop at the builder stage
docker build --target builder -t debug:builder .

# Shell into it
docker run -it debug:builder /bin/sh

# List what was built
docker run debug:builder ls -la /app/dist

# Check file sizes
docker run debug:builder du -sh /app/*
```

### Inspect Image Layers

```bash
# See layer sizes and commands
docker history myapp:latest

# Dive tool: interactive layer explorer
dive myapp:latest

# Export filesystem for inspection
docker create --name tmp myapp:latest
docker export tmp | tar tf - | head -50
docker rm tmp
```

### BuildKit Debug Tricks

- **Verbose output:** `docker build --progress=plain .` shows full build logs
- **Print in build:** Add `RUN ls -la /app && cat /app/config.json` temporarily
- **Debug image:** Use `distroless/static:debug` for a busybox shell in distroless
- **Buildx inspect:** `docker buildx du` shows cache usage

---

## 17 — Production Dockerfile Walkthrough

A complete, production-ready Node.js Dockerfile using every technique covered:

```dockerfile
# syntax=docker/dockerfile:1
ARG NODE_VERSION=20

# --- Stage 1: Dependencies ---
FROM node:${NODE_VERSION}-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

# --- Stage 2: Build ---
FROM node:${NODE_VERSION}-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY tsconfig.json ./
COPY src ./src
RUN npm run build && npm prune --omit=dev

# --- Stage 3: Production ---
FROM gcr.io/distroless/nodejs${NODE_VERSION}-debian12
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
ENV NODE_ENV=production
EXPOSE 3000
USER nonroot:nonroot
CMD ["dist/server.js"]
```

### What We Achieved

- BuildKit cache mounts
- Separate dep & build stages
- Distroless final image
- Non-root user

### Results

- ~130 MB final image
- No shell, no package manager
- Cached deps rebuild in seconds
- One file: dev, test, prod

---

## 18 — Summary & Further Reading

### Key Takeaways

- Use **named stages** (`FROM ... AS`) for clarity
- `COPY --from` only what your runtime needs
- Order layers from **least to most** frequently changing
- Choose the **smallest viable base** (scratch > distroless > alpine > slim)
- Enable **BuildKit** for cache mounts, secrets, and parallelism
- Use `--target` for **dev/test/prod** from one Dockerfile
- Always maintain a thorough `.dockerignore`

### Further Reading

- [Docker Docs: Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker BuildKit Documentation](https://docs.docker.com/build/buildkit/)
- [Distroless Container Images](https://github.com/GoogleContainerTools/distroless)
- [Dive: Image Layer Explorer](https://github.com/wagoodman/dive)
- [Docker Build Cache Guide](https://docs.docker.com/build/cache/)
- [Snyk: Node.js Docker Best Practices](https://snyk.io/blog/10-best-practices-to-containerize-nodejs-web-applications-with-docker/)

---

*Tags: `Multi-Stage` `BuildKit` `Optimization` `Images`*

*Thank you — questions?*
