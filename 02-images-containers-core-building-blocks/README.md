# Module 2 Notes - Docker Images & Containers: The Core Building Blocks

Course: *Docker & Kubernetes: The Practical Guide*

Core focus of this module: understanding **images** and **containers** as two distinct, related concepts - and how to build custom images instead of just running pre-built ones.

## The example project

A small Node.js app (Express + body-parser):

- `server.js` - web server listening on port 80. `GET /` returns HTML; `POST /store-goal` reads a `goal` value from the request body, stores it, and redirects back to `/`, rendering the stored goal.
- `public/` - static CSS served by the app.
- `package.json` - declares the app's dependencies (this is a Node-specific concept, not a Docker one).

Running it *without* Docker would require installing Node.js locally, running `npm install`, then `node server.js`. The whole point of this module is doing this **inside a container** instead.

### Snippet folders (progression through the module)

All under `snippets/`:

- `snippets/nodejs-app-first-dockerfile/` - the first working `Dockerfile`, written the naive way (`COPY . .` before `RUN npm install`), so every rebuild reinstalls dependencies even if only app code changed.
- `snippets/nodejs-app-finished/` - the optimized, final version: `package.json` copied and `npm install` run *before* the rest of the code, so dependency install is cached across rebuilds unless `package.json` itself changes. Also pins `FROM node:12` instead of floating `node`.
- `snippets/python-app-finished/` - the same core idea (base image + your code + `Dockerfile`) applied to a small Python script instead of Node, to reinforce that the pattern is language-agnostic.

## Building a custom image

- Most real use cases don't just run an existing image (like the bare `node` image) - you build **on top of** a base image to add your own code.
- Pattern: pull an official base image for your language/runtime (`node`, `python`, `php`, `golang`, ...) → copy your app code and dependencies into it → describe all of this in a `Dockerfile`.
- This pattern is language-agnostic - the same approach applies regardless of what the app is written in.

## Images are read-only once built

This is the most important conceptual point of the module:

- A `Dockerfile`'s `COPY` instructions take a **snapshot** of your source code at build time.
- Editing your source code afterward has **zero effect** on an already-built image, or on any container already running from it - the image doesn't "watch" the folder, it only copied what existed at build time.
- To pick up code changes, you must **rebuild** the image (`docker build .` again). This produces a technically new image, even if it looks "the same" to you.
- Why this matters: images are meant to be a locked, frozen template/blueprint - not something that mutates behind your back. (A faster workflow for picking up live code changes exists and is covered later in the course.)

## Image vs. Container - the mental model

- **Image** = the template/blueprint. Contains your code + everything needed to run it (dependencies, runtime environment). Built from a `Dockerfile`.
- **Container** = a running instance based on an image. It's a thin extra layer on top of the image (not a copy of it) - this is what's actually executing, isolated and independent from other containers.
- Multiple containers can run from the **same** image without duplicating the underlying code/environment for each one - Docker reuses the image's layers across all containers built from it. This is what makes containers lightweight compared to VMs.
- Bottom line: Docker is fundamentally about **containers** (that's what runs your app) - images just exist to make containers possible and reproducible.

## Key commands recap

```bash
docker build .                        # build a custom image from a Dockerfile
docker run -p <host_port>:<container_port> <image_id>   # run a container from an image
docker ps                             # list running containers
docker stop <container_id_or_name>    # stop a running container
```
