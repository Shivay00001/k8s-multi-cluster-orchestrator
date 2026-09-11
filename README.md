# Shivay00001/k8s-multi-cluster-orchestrator

A lightweight, high-performance Go HTTP service designed as the foundational runtime for a Kubernetes multi-cluster orchestration layer. The service exposes a real-time operational status endpoint and ships as a minimal, container-ready binary built on Go 1.20.

## 🚀 Overview

This repository contains a Go-based microservice that serves as the core runtime component of a multi-cluster orchestration system. At its current stage, the service implements a zero-dependency HTTP server (using only the Go standard library) that reports live operational status — the heartbeat upon which cluster health monitoring, scheduling, and orchestration logic can be layered.

The project is engineered for container-first deployment: a single `Dockerfile` produces a self-contained image that runs identically on any laptop, VM, or Kubernetes node.

## ✨ Features

- **Zero external dependencies** — built entirely on the Go standard library (`net/http`, `log`, `time`, `fmt`), meaning no supply-chain risk and reproducible builds via `go.mod` alone.
- **Real-time operational heartbeat** — the root endpoint returns the current server timestamp, enabling liveness/readiness probing and clock-drift verification across clusters.
- **High-performance HTTP listener** — Go's concurrent `net/http` server handles thousands of simultaneous connections on port `8080` with minimal memory footprint.
- **Container-native** — single-stage `golang:1.20-alpine` build produces a compact, portable image.
- **Fail-fast startup** — `log.Fatal` on server binding ensures the process exits loudly if the port is unavailable, which integrates cleanly with container restart policies and Kubernetes probes.

## 🏗️ Architecture / How It Works

The codebase is intentionally minimal and consists of a single entrypoint, `main.go`:

```
┌─────────────────────────────────────────────────────┐
│                    main.go                           │
│                                                      │
│  1. package main                                     │
│  2. http.HandleFunc("/", handler)   ← registers the  │
│     root route on the DefaultServeMux                │
│  3. Handler writes:                                  │
│     "System Operational: <current timestamp>"        │
│     using time.Now() at request time                 │
│  4. log.Println(...) announces startup               │
│  5. log.Fatal(http.ListenAndServe(":8080", nil))     │
│     binds port 8080 and blocks forever; any bind     │
│     error terminates the process with a logged cause │
└─────────────────────────────────────────────────────┘
```

**Request lifecycle:**

1. An HTTP client (load balancer, Kubernetes probe, or browser) sends any request to `/` on port `8080`.
2. The registered handler on Go's `DefaultServeMux` is invoked concurrently in its own goroutine.
3. The handler evaluates `time.Now()` at request time and writes `System Operational: 2006-01-02 15:04:05 ...` to the response body.
4. The connection is handled and recycled by Go's built-in HTTP server — no external router, framework, or middleware is involved.

**Module identity:** the Go module is declared as `github.com/Shivay00001/k8s-multi-cluster-orchestrator` (`go.mod`, Go 1.20), so the compiled binary is fully self-contained.

**Container build (`Dockerfile`):**

1. Base image `golang:1.20-alpine` is pulled.
2. Working directory is set to `/app` and the full repository is copied in.
3. `go build -o app` compiles a static binary named `app`.
4. At container start, `CMD ["./app"]` launches the HTTP server on port `8080`.

## 🐳 Docker Deployment (Run Anywhere)

The repository ships with a ready-to-use `Dockerfile`. No other dependencies are required beyond Docker itself.

### 1. Build the image

```bash
docker build -t shivay00001/k8s-multi-cluster-orchestrator .
```

### 2. Run the container

```bash
docker run -d --name orchestrator -p 8080:8080 shivay00001/k8s-multi-cluster-orchestrator
```

### 3. Verify the service

```bash
curl http://localhost:8080/
# Expected output:
# System Operational: 2026-01-01 12:00:00.000000000 +0000 UTC
```

### 4. Stop and clean up

```bash
docker stop orchestrator && docker rm orchestrator
```

> **Note on `docker-compose`:** this project does not currently include a `docker-compose.yml`. The single-service architecture is fully covered by the `docker build` / `docker run` flow above. If you prefer Compose, a minimal file would be:
>
> ```yaml
> services:
>   orchestrator:
>     build: .
>     ports:
>       - "8080:8080"
> ```
> then run `docker-compose up -d --build`.

## 🛠️ Local Development (Without Docker)

Requires Go 1.20+:

```bash
# Build the binary
go build -o app

# Run the service
./app
# Output: Starting high-performance service on :8080
```

Then visit `http://localhost:8080/` to see the live operational timestamp.

## 🔒 Security & Secrets Hygiene

The `.gitignore` is configured to exclude sensitive material from version control, including `.env` files, `*.key`, `credentials.json`, `service-account.json`, and other credential artifacts. Never commit cluster kubeconfigs, cloud provider keys, or API tokens to this repository.

## 📄 License

This project is distributed under the **VisionQuantech Custom Commercial License**:

- **Personal / educational / non-earning use** — free.
- **Personal revenue-generating use** — requires a 15–30% revenue share on gross earnings derived from the Software.
- **Business / enterprise use** — requires a separate commercial license. Contact: **visionquantech@proton.me**

See the [LICENSE](LICENSE) file for full terms. The Software is provided "AS IS", without warranty of any kind.