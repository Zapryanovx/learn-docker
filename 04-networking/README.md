# Module 4 Notes - Networking: (Cross-)Container Communication

Course: *Docker & Kubernetes: The Practical Guide*

Core focus of this module: how an app inside a container sends requests **out** - to the internet, to the host machine, and to other containers. The key fact behind all of it: inside a container, `localhost` means the container itself.

## The example project (`snippets/favorites-app/`)

A Node web API (Express, Axios, Mongoose) that returns JSON, tested with Postman:
- `GET /movies`, `GET /people` - fetch data from the public Star Wars API (`swapi.dev`) and return it. Nothing is stored.
- `POST /favorites`, `GET /favorites` - save / list your favorite movies and characters in a **MongoDB** database.

So the catalog comes from someone else's API, and only the user's own data (favorites) goes into our database - like a streaming service's catalog vs. your personal "My List". One app, two outgoing connections, which makes it a good demo for networking.

## Three kinds of outgoing communication

| Scenario | Example | Works out of the box? | Address to use inside the container |
|---|---|---|---|
| Container → internet | `swapi.dev` | **yes** | normal address |
| Container → host machine | MongoDB installed on your PC | no, `localhost` fails | `host.docker.internal` |
| Container → other container | MongoDB in its own container | no | container name, on a shared Docker network |

The isolation from earlier modules is mostly about **incoming** traffic (opened with `-p`). Outgoing requests to the internet just work - like a home PC that can open any website, while nobody outside can reach it unless a router port is opened.

## Container → internet

No setup at all. `axios.get('https://swapi.dev/api/films')` works the same inside a container as outside.

## Container → host machine

```
localhost on the host        -> your PC         (MongoDB is here)
localhost in the container   -> the container   (nothing is here)
```
`localhost` always means "myself", and a container has its own network namespace, so its own `localhost`. Hence `mongodb://localhost:27017/...` gives `ECONNREFUSED 127.0.0.1:27017` once the app runs in a container.

Fix: the special name **`host.docker.internal`**, which Docker resolves to the host's address:
```js
mongoose.connect('mongodb://host.docker.internal:27017/swfavorites', ...)
```
- Hardcoding the host's IP isn't an option: it changes, it's different on every machine, and on Windows the containers live in the WSL2 VM, where the host has a different address than `ipconfig` shows.
- Works out of the box on Docker Desktop (Windows/macOS). On Linux add `--add-host=host.docker.internal:host-gateway` to `docker run`.
- Hardcoding it breaks running the app outside Docker. In a real project the URL comes from an env var (module 3): `process.env.MONGODB_URL || 'mongodb://localhost:27017/swfavorites'`, and `docker run -e MONGODB_URL=...` sets the Docker value.

## Container → container

Best practice: **one container does one main thing.** The Node API and MongoDB go into separate containers:
- update/rebuild the API without touching the database
- MongoDB has an official image (`mongo`) - no Dockerfile needed
- scale the API independently (5 API containers, 1 database)
- separate logs and lifecycles

```bash
docker run -d --name mongodb mongo
```

### Option 1: the container's IP (works, but bad)
```bash
docker container inspect mongodb     # NetworkSettings -> IPAddress, e.g. 172.17.0.2
```
```js
mongoose.connect('mongodb://172.17.0.2:27017/swfavorites', ...)
```
The IP can change with every new container, so the code (and the image) would need constant changes.

### Option 2: a Docker network (the right way)
```bash
docker network create favorites-net
docker run -d --name mongodb --network favorites-net mongo
docker run -d --rm --name favorites --network favorites-net -p 3000:3000 favorites-node
```
```js
mongoose.connect('mongodb://mongodb:27017/swfavorites', ...)   // container name as the host
```
- Containers in the same network reach each other **by container name** - Docker acts like a built-in DNS.
- Networks must be created explicitly with `docker network create` (unlike volumes, `--network` won't create one).
- `mongodb` needs **no `-p`**: `-p` is only for traffic from the host into a container. Container-to-container traffic inside a network uses the container's own ports directly.
- Name resolution only works in **user-defined** networks. Containers started without `--network` land in Docker's default `bridge` network, where they only see each other by IP.

## How Docker resolves these names

Docker does **not** rewrite your source code. It owns the environment the container runs in, so when a request **leaves the container**, Docker sees the target (`mongodb`, `host.docker.internal`) and resolves it to the real IP at that moment.

Consequence: this only works for requests sent **from inside a container**. JavaScript running in a user's browser (e.g. a frontend calling an API) runs outside Docker, so a container name in browser-side code means nothing - the browser has to use `localhost:<published port>`.

## Network drivers

`docker network create --driver <driver> name` - the default is `bridge`, which is what this module uses and what you'll need in almost every case.

| Driver | What it does | When |
|---|---|---|
| `bridge` (default) | virtual network on one machine, containers find each other by name, outside access only via `-p` | almost always |
| `host` | no separate network: the container uses the host's network directly, `-p` not needed | rare. On Docker Desktop the "host" is the Linux VM, not your PC |
| `overlay` | connects containers across multiple Docker hosts | Docker Swarm (dated). Kubernetes solves this its own way |
| `macvlan` | gives the container its own MAC address, appears as a physical device on the LAN | special cases |
| `none` | networking disabled | full isolation |

Analogy: `bridge` = flats in one building with an intercom between neighbours, the front door (`-p`) leads outside. `host` = no flat, living in the hallway. `none` = a room with no doors.

## Gotchas I ran into

- `ECONNREFUSED 127.0.0.1:3000` in Postman while running `node app.js` locally - there was no MongoDB, and `app.listen(3000)` only runs inside the successful `mongoose.connect` callback. So the server never started, even for `/movies`. Starting the database afterwards doesn't help: Node had already given up, the app has to be restarted.
- No local MongoDB? `docker run -d --name mongodb -p 27017:27017 mongo` - with the port published, it looks exactly like a local install to `node app.js`.
- Commented the DB connection out in `app.js`, but the container still tried to connect - the image was built **before** the edit. Image = snapshot, rebuild after every code change (no bind mount here).
- `docker run -d --rm ...` → `docker ps` empty. The app couldn't reach the database, Node exited, the container stopped and `--rm` deleted it along with its logs. Debug by running without `-d` (error prints in the terminal) or without `--rm` (then `docker logs`).
- The `mongodb` container had stopped in the meantime → `docker start mongodb` (it still existed, so `start`, not `run`).
- `npm install` on the 2020 project shows deprecation and vulnerability warnings. Don't run `npm audit fix --force` - it can jump major versions (e.g. mongoose 5 → 8) and break the course code.
- Old mongoose prints `Current Server Discovery and Monitoring engine is deprecated` - just a warning, not the error.

## Key commands (cheat sheet)

```bash
docker network create NAME                # create a (bridge) network
docker network create --driver DRIVER NAME
docker network ls                         # list networks
docker network inspect NAME               # details, connected containers
docker network rm NAME                    # remove a network
docker network prune                      # remove unused networks

docker run --network NAME IMAGE           # start a container inside a network
docker container inspect CONTAINER        # details incl. NetworkSettings / IPAddress
docker start CONTAINER                    # restart a stopped (not removed) container
docker logs CONTAINER                     # see why a container stopped
```

Coming next: the MongoDB data is still stored in the container layer and is lost when the `mongodb` container is removed - solved with a volume in the multi-container module.
