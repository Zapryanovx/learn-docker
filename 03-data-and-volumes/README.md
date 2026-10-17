# Module 3 Notes - Managing Data & Working with Volumes

Course: *Docker & Kubernetes: The Practical Guide*

Core focus of this module: containers can read and write data, but that data dies with the container. **Volumes** and **bind mounts** connect container folders to the host so data survives, and so code changes reach the container without rebuilding. The module ends with **ARG** and **ENV** for making images and containers configurable.

## Three kinds of data

| Kind | Example | Read/write | Where it lives |
|---|---|---|---|
| Application code + environment | `server.js`, Node runtime, `node_modules` | read-only | **image** (copied in at build time) |
| Temporary app data | text entered in a form, files in `temp/` | read-write | **container layer** - fine to lose |
| Permanent app data | user accounts, saved feedback | read-write | **volume** - must survive |

The running app should never edit its own source code - that's why code belongs in the read-only image.

## The example project (`snippets/feedback-app/`)

A Node/Express feedback form. On submit, `server.js` writes the feedback to `temp/<title>.txt`, then copies it to `feedback/<title>.txt` if that file doesn't exist yet. So the app has all three kinds of data: code, temporary files (`temp/`) and permanent files (`feedback/`). See the snippet's README for the Dockerfile and the full set of `docker run` commands.

## The container layer, and why data gets lost

- A container = the read-only image layers + one thin **read-write layer** on top.
- The container layer doesn't copy the image's files. Reads fall through to the image; writes go only into the container layer. Changing an existing image file copies just that file up first (**copy-on-write**). The image itself is never modified.
- Mental model: lookup order is *mounts → container layer → image*.
- `docker stop` + `docker start` keeps the container layer - data is still there.
- `docker rm` (or `--rm`) deletes the container layer - data is gone. A new container from the same image starts clean.
- In practice containers get removed all the time (every code change = new image = new container), so anything important can't live in the container layer.
- Files written inside the container do **not** appear in your project folder on the host. `COPY` was a one-time snapshot, there's no ongoing connection.

## Volumes

- **Volumes are folders on the host machine, managed by Docker, mounted into a container path.**
- Mounting is **not copying**: it's the same storage seen under two paths (like two references to the same object). Changes show up on both sides instantly.
- Unlike `COPY` (one-time snapshot at build), a mount is a live connection at run time.
- The container never "owned" the data - removing the container removes the mount, not the folder.

### Anonymous volumes
```dockerfile
VOLUME [ "/app/feedback" ]        # in the Dockerfile
```
```bash
docker run -v /app/feedback ...   # or on the command line
```
- Docker picks a random name. Tied to one container.
- Removed automatically **only** if the container was started with `--rm`. Without `--rm`, `docker rm` leaves it behind - but a new container gets a *new* anonymous volume anyway, so the old one is just orphaned (clean up with `docker volume prune` or `docker rm -v CONTAINER`).
- **Not** useful for persisting data. Useful for: protecting a sub-folder from being hidden by a bind mount (`node_modules`), and outsourcing temporary data (`temp/`) from the container layer to the host for a bit of efficiency.

### Named volumes
```bash
docker run -v feedback:/app/feedback ...
```
- `name:/container/path`. Can't be created in a Dockerfile, only at `docker run` (or `docker volume create`).
- Not tied to any container - survive `docker rm` and `--rm`. A new container with the same `-v name:...` sees the old data.
- Can be shared by several containers at once (watch out for concurrent writes).
- The right tool for permanent data, also in production.
- Not meant to be browsed/edited by hand - access it through a container.

### Side note: `rename` across a volume
Moving a file from `temp/` into a volume-backed `feedback/` with Node's `fs.rename` fails with `EXDEV: cross-device link not permitted` - from the kernel's point of view the volume is a different device. Fix: `fs.copyFile` + `fs.unlink`.

## Bind mounts

```bash
docker run -v "C:\full\path\to\project:/app" ...
docker run -v "${PWD}:/app" ...       # PowerShell shortcut for the current folder
docker run -v "$(pwd):/app" ...       # bash / macOS / Linux
```
- Same mechanism, but **you** choose the host folder (absolute path; quote it if it has spaces).
- Purpose: during development, edit code on the host and the container sees it immediately - no rebuild per change.
- The code still lives in one place (your project). This is *not* the same as editing code inside the container, which would be lost and never reach the project.
- Docker never deletes a bind-mounted folder. Not shown in `docker volume ls`.
- **Development only.** In production there's no bind mount - code is baked into the image with `COPY`.

### How Docker tells them apart (left side of `-v`)

| `-v` form | Type |
|---|---|
| `-v /app/data` | anonymous volume |
| `-v data:/app/data` | named volume |
| `-v /absolute/host/path:/app/data` | bind mount |

Same rule as `-p`: **left = outside, right = inside the container.**

## Combining mounts: the longer path wins

Binding the project onto `/app` hides everything the image had in `/app` - including `node_modules` created by `RUN npm install`. The project folder on the host has no `node_modules`, so the app crashes (`Cannot find module 'express'`).

Fix: add a more specific mount. When mounts overlap, **the longer (more specific) container path wins** - it *hides* the parent mount for that sub-folder, it doesn't delete anything.

```bash
-v "${PWD}:/app"         # /app            -> your project
-v /app/node_modules     # /app/node_modules -> anonymous volume, filled from the image
-v feedback:/app/feedback  # /app/feedback -> named volume
```

Resulting view inside the container:
```
/app
├── server.js, pages/, public/   <- host (bind mount)
├── node_modules/                <- image (anonymous volume)
└── feedback/                    <- Docker named volume
```
The code that `COPY . .` put into the image is not used at all in this setup - the bind mount hides it.

## Seeing code changes: nodemon

A bind mount puts the new file in the container, but Node read `server.js` once at start-up and runs from memory. Files read per request (e.g. HTML sent with `res.sendFile`) update immediately; the server code needs a restart.

- `nodemon` watches files and restarts Node automatically: `"start": "nodemon server.js"` in `package.json`, `CMD ["npm", "start"]` in the Dockerfile.
- `npm start` → `package.json` `scripts.start` → `nodemon` → `node server.js`. Keeping the start command in `package.json` means changing it needs only a container restart (the file comes through the bind mount), not a rebuild.
- **Windows (WSL2) gotcha:** file-change events from the Windows filesystem don't reach the Linux VM, so nodemon never notices. Use polling: `nodemon -L server.js`.
- Rebuild is still needed when the Dockerfile or the dependencies in `package.json` change.

## Read-only volumes

```bash
-v "${PWD}:/app:ro"
```
- `:ro` after the container path makes the mount read-only **for the container** (you can still edit on the host). Enforces and documents that the app must not change its code.
- Applies to all sub-folders too, so every folder the app writes to needs its own (writable) mount, which wins because the path is longer:
  - `/app/feedback` → already a named volume
  - `/app/temp` → add an anonymous volume `-v /app/temp`
- Put that anonymous volume on `docker run`, not as `VOLUME` in the Dockerfile - it exists because of the bind mount, which is also a run-time decision.

## Summary: which mount for what

| | Anonymous volume | Named volume | Bind mount |
|---|---|---|---|
| Host location | Docker-managed, random name | Docker-managed, your name | your folder |
| Survives `docker rm` | no (with `--rm`) / orphaned (without) | yes | yes (it's your folder) |
| Shared across containers | no | yes | yes |
| Created in Dockerfile | yes (`VOLUME`) | no | no |
| Use for | protecting sub-folders, temp data | permanent data | live source code in development |

## Managing volumes

```bash
docker volume ls                    # all Docker-managed volumes (bind mounts not listed)
docker volume create feedback-files # create manually (usually unnecessary, -v creates it)
docker volume inspect feedback      # details: Mountpoint, read-only options, ...
docker volume rm feedback           # delete - ALL DATA IN IT IS GONE, no undo
docker volume prune                 # delete unused anonymous volumes (-a: also unused named ones)
```
- A volume in use by a container (even a stopped one) can't be removed - remove the container first.
- `Mountpoint` (e.g. `/var/lib/docker/volumes/feedback/_data`) is a path inside Docker Desktop's Linux VM (WSL2 on Windows), whose disk is a single `ext4.vhdx` file - which is why you won't find it in Explorer.
- To look inside a volume, go through a container: `docker run --rm -v feedback:/data alpine ls /data`.

## COPY vs bind mounts

If the bind mount hides `/app` anyway, why keep `COPY . .`? Because the bind mount is only for development. Production containers run without it and need the code snapshot inside the image. Keep `COPY`, add the bind mount only on the dev `docker run`.

## `.dockerignore`

Works like `.gitignore`, but for `COPY`. Keeps things out of the build context and the image:
```
node_modules
Dockerfile
.git
.env
```
- `node_modules` - installed inside the image by `RUN npm install`, a local copy would overwrite it.
- `Dockerfile` - the image doesn't need it, and editing it would otherwise invalidate the `COPY . .` layer.
- `.env` - secrets must not be baked into the image.

`.gitignore` keeps secrets out of git, `.dockerignore` keeps them out of the image. Both are needed.

## ENV - runtime environment variables

```dockerfile
ENV PORT=80          # default value (use key=value, the old "ENV key value" form is deprecated)
EXPOSE $PORT
```
```js
app.listen(process.env.PORT);   // Java equivalent: System.getenv("PORT")
```
```bash
docker run --env PORT=8000 -p 3000:8000 ...    # override, -e for short
docker run --env-file ./.env -p 3000:8000 ...  # read KEY=VALUE lines from a file
```
- Available in the Dockerfile **and** in the running app. Overridable per `docker run` without rebuilding.
- Point: **one image, different config per environment** (dev/prod DB URL, ports, log level). Code is the same everywhere, config comes from outside.
- Variables from `--env-file` don't need to be declared in the Dockerfile. The file is read by the CLI on the host, it isn't copied into the container.

### Security
- Anything in the Dockerfile is baked into the image - `docker history <image>` shows it. **Never put passwords/keys in `ENV` or `ARG`.**
- Pass secrets at runtime with `--env-file`, and never commit that file (`.env` in `.gitignore` and `.dockerignore`).
- Non-secret defaults (like `PORT`) in the Dockerfile are fine.

## ARG - build-time arguments

```dockerfile
ARG DEFAULT_PORT=80
ENV PORT=$DEFAULT_PORT
```
```bash
docker build -t feedback-node:web-app .
docker build -t feedback-node:dev --build-arg DEFAULT_PORT=8000 .
```
- Exists **only during `docker build`**. Not visible to `CMD` or the app - if the app needs the value, pass it through an `ENV`.
- Lets you build different images from one unchanged Dockerfile (e.g. `ARG NODE_VERSION` + `FROM node:${NODE_VERSION}`; `ARG` is the only instruction allowed before `FROM`).
- Why not always ENV? There's no `docker build --env`: ENV can't be set from outside at build time, and it stays in every container even if it only mattered for the build.

| | ARG | ENV |
|---|---|---|
| Exists | during build only | during build + in the running container |
| Visible to app code | no | yes |
| Default set in Dockerfile | `ARG NAME=value` | `ENV NAME=value` |
| Overridden with | `docker build --build-arg` | `docker run -e` / `--env-file` |
| Changing it needs | a rebuild | just a new container |

### Placement matters for the cache
`ARG`/`ENV` lines are layers too. Worse: every `RUN` below an `ARG` gets the arg in its cache key - `docker history` literally shows `RUN |1 DEFAULT_PORT=8000 /bin/sh -c npm install`. With the `ARG` at the top, every `--build-arg` change re-ran `npm install`. Put `ARG`/`ENV` as low as possible, just above where they're used.

## Gotchas I ran into

- `docker build .` failed with `open Dockerfile: no such file or directory` - the `.` is the current folder, run it from the folder that contains the Dockerfile. (VS Code's *Compact Folders* displays single-child folders merged on one line, which hid one level of nesting.)
- `docker rmi image .` - the `.` belongs to `docker build` only. `rmi` treated it as a second image name.
- `docker rmi` refused because a **stopped** container still referenced the image - `docker rm` it first (or use `--rm`). Don't use `rmi -f`.
- `open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified` - Docker Desktop isn't running, the CLI has no engine to talk to. Start it and wait for *Engine running* (`docker version` should show a Server section).
- `-p 3000:8000` then opening `localhost:8000` → connection refused. The browser uses the host (left) port: `localhost:3000`.
- Bind-mounted code changed but nothing happened: no `restarting due to changes` in `docker logs` → nodemon needs `-L` on Windows.

## Key commands (cheat sheet)

```bash
# volumes & mounts on docker run
docker run -v /app/data IMAGE                 # anonymous volume
docker run -v name:/app/data IMAGE            # named volume
docker run -v "/host/path:/app" IMAGE         # bind mount
docker run -v "/host/path:/app:ro" IMAGE      # read-only bind mount

# volume management
docker volume ls | create | inspect | rm | prune

# configuration
docker run -e KEY=value IMAGE                 # set an env var (--env)
docker run --env-file ./.env IMAGE            # env vars from a file
docker build --build-arg KEY=value .          # set a build arg

# misc used in this module
docker logs CONTAINER                         # output of a container
docker restart CONTAINER                      # stop + start
docker history IMAGE                          # layers + the instructions that made them
```
