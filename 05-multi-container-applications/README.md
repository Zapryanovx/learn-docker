# Module 5 Notes - Building Multi-Container Applications with Docker

Course: *Docker & Kubernetes: The Practical Guide*

Core focus of this module: no new Docker features - instead, everything from modules 1-4 (images, volumes, bind mounts, ENV, networks) combined in a realistic app made of three services, each in its own container. It's a **development** setup: live code reload, not production-ready.

## The target app (`snippets/goals-app/`)

| Building block | Tech | Requirements when containerized |
|---|---|---|
| Database | MongoDB (official `mongo` image) | data must persist, access restricted with user/password |
| Backend | Node/Express REST API (JSON only, no HTML) | log files must persist, live code updates |
| Frontend | React single page application | live code updates |

The frontend is what users see. It talks to the backend, which talks to the database. The frontend never talks to the database directly.

See the snippet README for the Dockerfiles and the full set of `docker run` commands.

## Database container

```bash
docker run --name mongodb --rm -d --network goals-net \
  -v goals-data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=max -e MONGO_INITDB_ROOT_PASSWORD=secret \
  mongo
```
- No Dockerfile needed - the official image already is "MongoDB in a container".
- **Persistence:** MongoDB stores its data in `/data/db` inside the container (documented on Docker Hub). A named volume on that path keeps the data when the container is removed. Without it (module 4), every `docker rm` wiped the database.
- **Authentication:** the image supports `MONGO_INITDB_ROOT_USERNAME` / `MONGO_INITDB_ROOT_PASSWORD`. The client then connects with `user:password@host` and `?authSource=admin` (the root user lives in the `admin` database).
- **No `-p`**: only the backend container talks to it, inside the network.

### Gotcha: `MONGO_INITDB_*` only work on an empty data folder
The user is created **only the first time** the database is initialized. If the volume already contains a database (here: an old volume named `data` from earlier experiments), MongoDB just opens it and ignores the variables - the user never exists and the backend gets `AuthenticationFailed`. Fix: a fresh named volume (`goals-data`). Changing the credentials later also needs a fresh volume (or changing the user inside Mongo).

## Backend container

```bash
docker run --name goals-backend --rm -d -p 80:80 --network goals-net \
  -v "${PWD}:/app" -v logs:/app/logs -v /app/node_modules \
  -e MONGODB_USERNAME=max \
  goals-node
```
Every flag is something from an earlier module:

| Part | From | Why |
|---|---|---|
| `--network goals-net` | module 4 | reach MongoDB as `mongodb:27017` |
| `-p 80:80` | module 1 | the React code in the **browser** calls `http://localhost/goals` |
| `-v "${PWD}:/app"` | module 3 | bind mount: code changes reach the container without rebuilding |
| `-v logs:/app/logs` | module 3 | named volume: `access.log` survives container removal |
| `-v /app/node_modules` | module 3 | anonymous volume: protects the image's `node_modules` from the bind mount (longer path wins) |
| `-e MONGODB_USERNAME=max` | module 3 | ENV: credentials are config, not code |
| `nodemon -L` in `npm start` | module 3 | restart Node on change, polling because of WSL2 |

Connection string in `app.js`:
```js
`mongodb://${process.env.MONGODB_USERNAME}:${process.env.MONGODB_PASSWORD}@mongodb:27017/course-goals?authSource=admin`
```
Defaults (`root` / `secret`) sit in the Dockerfile as `ENV`, overridden with `-e`. Fine for a course demo, but real credentials don't belong in a Dockerfile (`docker history` shows them) - use `--env-file`.

How the backend got there step by step: `localhost:27017` failed inside the container → `host.docker.internal` (Mongo published on the host) worked → final version: both in `goals-net`, Mongo by container name, no published Mongo port.

## Frontend container

```bash
docker run --name goals-frontend --rm -d -it -p 3000:3000 \
  -v "${PWD}/src:/app/src" -e WATCHPACK_POLLING=true \
  goals-react
```

### Why `FROM node` for a React app?
The React code runs in the **browser**. What runs in the container is `react-scripts start`: a Node program that compiles `src/` (JSX, modern JS) into plain JavaScript and serves it on port 3000 with a dev server. The browser downloads the files from there and runs React itself. So the container needs Node for the tooling, not for the app. (In production you'd `npm run build` to static files and serve them without Node - covered later.)

### Why it isn't in the network
Docker resolves container names only for requests that **leave a container** (module 4). The frontend container only serves files - the actual `fetch('http://localhost/goals')` runs in the browser, outside Docker. So:
- the backend URL in React code must be `localhost` + the backend's **published** port, not `goals-backend`
- the backend needs `-p 80:80`
- the frontend container itself doesn't need `--network` at all

### Why `-it`
`react-scripts start` shuts itself down when stdin closes (so the dev server dies with its terminal). `docker run` gives a container a closed stdin by default, so the dev server exits immediately. `-i` keeps stdin open. (`-t` only adds a proper terminal: colours, formatting. Alternative: `-e CI=true`.)

### Live reload
- Bind mount only `src/` - the rest of the project (incl. `node_modules`) stays from the image, no anonymous volume needed.
- Like nodemon, webpack's file watcher gets no change events through WSL2. `react-scripts` has no `-L` flag; polling is switched on with an env var: `-e WATCHPACK_POLLING=true`. Env vars can't be added to a running container, so this needs a new `docker run`.

## Who needs `-p` and `--network`

| Container | `--network` | `-p` | Talked to by |
|---|---|---|---|
| mongodb | yes | no | backend container (inside the network) |
| goals-backend | yes | `80:80` | browser (outside Docker) |
| goals-frontend | no | `3000:3000` | browser (outside Docker) |

Rule: **`-p` whenever something outside Docker (browser, Postman, `node app.js` on the host) connects to a container.** The left number is the port on your machine. Container-to-container inside a network needs no `-p`.

## Development vs. production

Everything here is a dev setup: bind mounts, nodemon, the React dev server. A production setup has no live reload, code is baked into images, and the frontend is built into static files. Many developers use Docker only for development environments like this one.

## What's still annoying

Three long `docker run` commands, run one by one, each with volumes, env vars and network flags to remember. Forget one `-v` and something silently breaks. The next module fixes this with **Docker Compose**: one config file, one command to start everything.

## Gotchas I ran into

- **`Error: Unknown keyword formatMinimum`** when starting the frontend container. Three causes, fixed together:
  - no `.dockerignore`, so `COPY . .` copied the Windows `node_modules` over the one built by `npm install` - a mix of two installs
  - only `package.json` copied, not `package-lock.json`, so the container resolved newer dependency versions (`COPY package*.json .` copies both)
  - `FROM node` pulled Node 26, too new for `react-scripts 5` - pinned to `node:14`
- **Frontend container exits immediately without `-it`** - `react-scripts` stops when stdin is closed.
- **Backend `AuthenticationFailed`** - reused an old, non-empty volume, so `MONGO_INITDB_*` were ignored. New volume fixed it.
- **The "still failing" backend was an old container** - after fixing Mongo, `docker logs` still showed the old error because that backend had stopped before the fix. Without `--rm` the stopped container also blocks the name: `docker rm` it before re-running.
- **Typo in the bind mount path** (`...\backend\a:/app`) - Docker silently **created** an empty folder `a` on the host and mounted it over `/app`, hiding `app.js`. Quote the path, or use `"${PWD}:/app"` from the project folder.
- **Code changes not picked up** - no `restarting due to changes` in the logs. Backend: `nodemon -L`. Frontend: `-e WATCHPACK_POLLING=true`.
- `npm install` warnings: 108 vulnerabilities in old `react-scripts` dev tooling - don't `npm audit fix --force`. Newer npm also asks to approve `postinstall` scripts (`core-js` only prints a donation message, safe to leave unapproved).

## Key commands (cheat sheet)

```bash
docker network create goals-net
docker run ... --network goals-net ...      # put a container in the network
docker run ... -e KEY=value ...             # env var (image config, credentials)
docker run ... -it ...                      # keep stdin open (React dev server)
docker logs CONTAINER                       # why did it stop / did nodemon restart
docker restart CONTAINER                    # pick up package.json changes via bind mount
docker rename OLD NEW                       # rename a container (breaks name-based connections)
docker exec mongodb mongosh -u max -p secret --authenticationDatabase admin   # check the DB login directly
```
