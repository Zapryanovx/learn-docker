# goals-app - three containers working together

A "course goals" app: add, list and delete goals.

```
browser ──► frontend (React dev server, :3000)          serves the React code
browser ──► backend  (Node/Express REST API, :80)        React code calls http://localhost/goals
backend ──► mongodb  (official mongo image, :27017)      via the goals-net network, by container name
```

## backend/Dockerfile

```dockerfile
FROM node
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
EXPOSE 80
ENV MONGODB_USERNAME=root
ENV MONGODB_PASSWORD=secret
CMD ["npm", "start"]
```

- `npm start` → `nodemon -L app.js` (restarts on code changes, `-L` = polling for Windows/WSL2 bind mounts).
- `ENV MONGODB_USERNAME/PASSWORD` - defaults, overridden with `-e` on `docker run`. The course puts them in the Dockerfile for simplicity. For real credentials they'd come from `--env-file` instead (they're visible in `docker history`).
- `app.js` builds the connection string from them: `mongodb://${MONGODB_USERNAME}:${MONGODB_PASSWORD}@mongodb:27017/course-goals?authSource=admin`. `mongodb` = container name, resolved inside the network.
- The app writes request logs to `logs/access.log` (morgan) - persisted with a named volume.

## frontend/Dockerfile

```dockerfile
FROM node:14
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

- `npm start` → `react-scripts start`: compiles `src/` and serves it with a dev server on 3000. Node is needed for that tooling, the React code itself runs in the browser.
- Pinned to `node:14` - the unpinned `node` (v26) broke `react-scripts 5`.

Both projects have a `.dockerignore` with `node_modules`, `Dockerfile`, `.git` - without it, a local `node_modules` gets copied over the one `npm install` built in the image (`Error: Unknown keyword formatMinimum`).

## Run (development setup)

```bash
docker network create goals-net
```

**Database** - named volume for the data, root user created on first start:
```bash
docker run --name mongodb --rm -d --network goals-net \
  -v goals-data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=max -e MONGO_INITDB_ROOT_PASSWORD=secret \
  mongo
```

**Backend** - from `backend/`:
```bash
docker build -t goals-node .
docker run --name goals-backend --rm -d -p 80:80 --network goals-net \
  -v "${PWD}:/app" -v logs:/app/logs -v /app/node_modules \
  -e MONGODB_USERNAME=max \
  goals-node
```

**Frontend** - from `frontend/`:
```bash
docker build -t goals-react .
docker run --name goals-frontend --rm -d -it -p 3000:3000 \
  -v "${PWD}/src:/app/src" -e WATCHPACK_POLLING=true \
  goals-react
```

Open `localhost:3000`.

| Container | `--network` | `-p` | Why |
|---|---|---|---|
| mongodb | yes | no | only the backend talks to it, inside the network |
| goals-backend | yes | `80:80` | needs the network for mongodb, and the **browser** calls it |
| goals-frontend | no | `3000:3000` | only the browser talks to it. Its React code runs in the browser, outside Docker, so it can't use container names |
