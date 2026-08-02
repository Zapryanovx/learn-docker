# Module 1 Notes — Getting Started with Docker

Course: *Docker & Kubernetes: The Practical Guide*

## What is Docker?

- Docker is a **container technology** — a tool for creating and managing containers.
- A **container** = a standardized unit of software: your code + the dependencies/runtime it needs to run, all packaged together.
- Same container → same behavior, everywhere. No "works on my machine" surprises, because everything the app needs is baked in.
- Analogy: a picnic basket (food + dishes together) or a shipping container (self-contained, isolated goods that can move between any ship/truck).

## Why containers?

Three recurring problems containers solve:

1. **Dev vs. production mismatch** — e.g. code using a Node.js feature (top-level `await`) that only works on Node 14.3+. If the deployment server has an older Node version, the app breaks. Locking the exact runtime version inside the container removes this class of bug entirely.
2. **Team consistency** — different teammates with different locally-installed tool versions can end up with code that "works for me, not for you." A container guarantees everyone runs the exact same environment.
3. **Local project conflicts** — multiple projects needing different versions of the same runtime (e.g. Node 12 vs Node 14) normally forces constant uninstall/reinstall when switching projects. With containers, each project's version is locked inside its own container — switching projects = launching a different container, nothing to reinstall.

## Virtual Machines vs. Containers

| | Virtual Machines | Containers |
|---|---|---|
| What it virtualizes | A whole separate OS on top of the host | Just the app + what it needs, using the host OS's container support |
| Resource usage | Heavy — each VM duplicates a full OS | Lightweight — share the host kernel via the Docker Engine |
| Speed | Slower, more overhead | Fast, minimal overhead |
| Disk space | Large (redundant OS copies) | Small |
| Sharing/reproducing | Possible but clunky (no single shareable config) | Easy — Dockerfile + images are the shareable config |

Key point: **containers** are the core concept; **Docker** is just the (de-facto standard) tool for creating and managing them.

## Docker Setup

- **Docker Desktop** — recommended tool for macOS/Windows. Requires modern OS versions (e.g. Windows 10 Pro/Enterprise/Education, or Windows 10 Home with WSL2 enabled).
- **Docker Toolbox** — legacy fallback for older systems that can't run Docker Desktop; spins up a VirtualBox VM running Linux, since Docker itself needs a Linux kernel to run containers natively.
- **Linux** — no extra tool needed, the Docker Engine installs and runs natively.
- Docker must actually be **running** (whale icon in the tray/menu bar) before any `docker` command will work.

## Docker Tooling Overview

- **Docker Engine** — the core: a background daemon process + CLI. This is what actually builds/runs everything.
- **Docker Hub** — cloud registry for hosting and sharing images.
- **Docker Compose** — tool for managing multi-container projects more easily (covered in a later module).
- **Kubernetes** — separate tool (own module later) for managing/deploying complex containerized apps at scale.

## First Hands-On Example (this folder)

A minimal Node.js app (`app.mjs`) using top-level `await`, started via a pre-written `Dockerfile` (not yet built from scratch — that comes in the next module):

```dockerfile
FROM node
WORKDIR /app
COPY package.json /app
RUN npm install
COPY . /app
EXPOSE 3000
CMD ["node", "app.mjs"]
```

Commands used:

```bash
docker build .                    # build an image from the Dockerfile in the current dir
docker run -p 3000:3000 <image_id>  # run a container, map host port 3000 -> container port 3000
docker ps                         # list running containers
docker stop <container_name>      # stop a running container
```

**Takeaway:** Node.js was never installed locally, `npm install` was never run outside a container, yet the app runs at `localhost:3000` — proof the image bundles the entire runtime + app, and Docker is doing real work, not just a shortcut.
