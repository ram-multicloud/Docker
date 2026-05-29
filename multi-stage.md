# Docker Multi-Stage Builds

## Multi-Stage Build

A **multi-stage build** uses multiple `FROM` instructions in a single Dockerfile. Each `FROM` starts a new **stage** — a completely isolated build environment. You can selectively **copy artifacts** from one stage into another, leaving behind everything you don't need in the final image.

This means your final production image only contains what the application needs to *run*, not what it needed to *build*.

### Traditional (Single-Stage) Problem

```dockerfile
# ❌ Single-stage: build tools end up in production image
FROM node:20

WORKDIR /app
COPY package*.json ./
RUN npm install          # dev dependencies included
COPY . .
RUN npm run build

CMD ["node", "dist/index.js"]
```

**Result:** The image ships with Node.js source files, `node_modules` (all deps), build tools, and intermediate files — easily 500MB+.

---

## 2. How Multi-Stage Builds Work

```
┌──────────────────────┐      ┌──────────────────────┐
│   Stage 1: builder   │      │  Stage 2: production  │
│  (heavy base image)  │─────▶│   (slim base image)   │
│  Install deps        │ COPY │  Only final artifact  │
│  Compile/build       │      │  Runs the app         │
└──────────────────────┘      └──────────────────────┘
        Discarded                    Shipped
```

- Each stage is **isolated** — files don't leak between stages unless you explicitly `COPY --from=<stage>`.
- Earlier stages are **discarded** from the final image.
- You can have **2 or more stages**.

---

## 3. Use Cases

### 3.1 Compiled Languages (Go, Rust, Java, C++)

Compile in a full SDK image, copy only the binary to a minimal runtime image.

```dockerfile
# Go example
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o server .

FROM scratch                        # Empty base — just the binary!
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

**Before:** ~900MB | **After:** ~10MB

---

### 3.2 Frontend Applications (React, Vue, Angular)

Build assets with Node.js, serve with a lightweight web server.

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build                   # outputs to /app/dist

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
```

**Before:** ~1.2GB | **After:** ~25MB

---

### 3.3 Java / Spring Boot

Compile with Maven/Gradle, run on a slim JRE.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

### 3.4 Separating Test and Production

Run tests in one stage; only proceed to production image if tests pass.

```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

FROM base AS test
COPY requirements-dev.txt .
RUN pip install -r requirements-dev.txt
COPY . .
RUN pytest                          # Build fails here if tests fail

FROM base AS production
COPY . .
CMD ["gunicorn", "app:app"]
```

---

### 3.5 Security Scanning / Linting Stage

Lint code before building the final image.

```dockerfile
FROM node:20-alpine AS lint
WORKDIR /app
COPY . .
RUN npm run lint && npm run typecheck

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=lint /app/src ./src     # Only copy if lint passed

FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app .
CMD ["node", "src/index.js"]
```

---

## 4. Writing a Multi-Stage Dockerfile

### 4.1 Syntax Reference

```dockerfile
# Name a stage with AS <name>
FROM <image> AS <stage-name>

# Copy from a named stage
COPY --from=<stage-name> <src> <dest>

# Copy from a specific stage index (0-based)
COPY --from=0 <src> <dest>

# Copy from an external image (not a build stage)
COPY --from=nginx:alpine /etc/nginx/nginx.conf /etc/nginx/nginx.conf
```

### 4.2 Step-by-Step: Node.js REST API

```dockerfile
# ── Stage 1: Install ALL dependencies and build ──────────────────
FROM node:20-alpine AS builder

WORKDIR /app

# Copy manifests first (Docker layer cache optimization)
COPY package*.json ./
RUN npm ci                          # Clean install (respects package-lock.json)

COPY tsconfig.json ./
COPY src ./src
RUN npm run build                   # Compile TypeScript → dist/


# ── Stage 2: Production image ────────────────────────────────────
FROM node:20-alpine AS production

# Set non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

WORKDIR /app

# Copy ONLY production dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy compiled output from builder stage
COPY --from=builder /app/dist ./dist

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### 4.3 Building the Image

```bash
# Build normally (uses all stages, produces final stage)
docker build -t my-api:latest .

# Build and stop at a specific stage (useful for debugging)
docker build --target builder -t my-api:debug .

# Build with build args passed to stages
docker build --build-arg NODE_ENV=production -t my-api .
```

---

## 5. Best Practices

| Practice | Why it matters |
|---|---|
| Name your stages with `AS` | Makes `COPY --from` readable and maintainable |
| Copy `package.json` before source | Maximizes Docker layer cache for dependencies |
| Use slim/alpine base images | Reduces attack surface and image size |
| Use `--only=production` for npm | Excludes dev dependencies from final image |
| Run as non-root user | Security hardening |
| Use `.dockerignore` | Prevents local `node_modules`, `.git`, etc. from being copied into build context |

### Example `.dockerignore`

```
node_modules
.git
.env
*.log
dist
coverage
```

---

## 6. Debugging Tips

**Inspect an intermediate stage:**
```bash
# Build up to the 'builder' stage only
docker build --target builder -t debug-build .

# Shell into it
docker run --rm -it debug-build sh
```

**Check final image size:**
```bash
docker images my-api
```

**See all layers and their sizes:**
```bash
docker history my-api:latest
```

---

## 7. Quick Reference Card

```
Dockerfile Anatomy — Multi-Stage

FROM image AS stage1          ← Start stage, give it a name
  RUN ...                     ← Heavy build steps
  COPY ...                    ←
  RUN compile...              ←

FROM slim-image AS stage2     ← Start a new, clean stage
  COPY --from=stage1 /x /y   ← Pull only what you need
  CMD [...]                   ← This is what gets shipped
```

---

## Summary

Multi-stage builds solve two key problems:

1. **Image size** — Final images only contain runtime artifacts, not build tools.
2. **Security** — Fewer packages = smaller attack surface.

They are now the standard approach for any production Docker image involving a build step.
