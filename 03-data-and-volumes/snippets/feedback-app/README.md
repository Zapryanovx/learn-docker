# feedback-app - what the Dockerfile does

```dockerfile
FROM node:14
WORKDIR /app
COPY package.json /app/
RUN npm install
COPY . .
ARG DEFAULT_PORT=80
ENV PORT=$DEFAULT_PORT
EXPOSE $PORT
CMD [ "npm", "start" ]
```

- `FROM node:14` / `WORKDIR` / `COPY package.json` / `RUN npm install` / `COPY . .` - same cached layer order as module 2.
- `ARG DEFAULT_PORT=80` - build-time argument, only exists during `docker build`. Placed **after** `RUN npm install` on purpose: every `RUN` below an `ARG` gets that value as part of its cache key, so changing the arg would otherwise re-run `npm install` for nothing.
- `ENV PORT=$DEFAULT_PORT` - runtime environment variable, read by the app via `process.env.PORT`. Default comes from the build arg, can be overridden on `docker run`.
- `EXPOSE $PORT` - documents the port.
- `CMD [ "npm", "start" ]` - runs the `start` script from `package.json`, which is `nodemon -L server.js` (restarts Node when files change, `-L` = polling, needed on Windows/WSL2 bind mounts where file-change events don't arrive).

`.dockerignore` keeps `node_modules`, `Dockerfile`, `.git` and `.env` out of `COPY . .` (secrets must not end up in the image).

The app itself: a feedback form. On submit, `server.js` writes the text to `temp/<title>.txt`, then copies it to `feedback/<title>.txt` if it doesn't exist yet (otherwise redirects to `/exists`). Saved files can be viewed at `/feedback/<title>.txt`.

## Build

```bash
docker build -t feedback-node:web-app .                                   # DEFAULT_PORT = 80
docker build -t feedback-node:dev --build-arg DEFAULT_PORT=8000 .         # same Dockerfile, different default
```

## Run - progression through the module

**1. No volumes** - feedback is lost when the container is removed:
```bash
docker run -d --rm -p 3000:80 --name feedback-app feedback-node:web-app
```

**2. Named volume** - feedback survives container removal:
```bash
docker run -d --rm -p 3000:80 --name feedback-app -v feedback:/app/feedback feedback-node:web-app
```

**3. Development setup** - named volume + read-only bind mount for live code + anonymous volumes to protect / allow writes to sub-folders:
```bash
docker run -d --rm -p 3000:80 --name feedback-app \
  -v feedback:/app/feedback \
  -v "${PWD}:/app:ro" \
  -v /app/temp \
  -v /app/node_modules \
  feedback-node:web-app
```
(`${PWD}` in PowerShell, `$(pwd)` in bash. Run from this folder.)

**4. Environment variables at runtime:**
```bash
docker run -d --rm -p 3000:8000 --env PORT=8000 --name feedback-app feedback-node:web-app
cp .env.example .env
docker run -d --rm -p 3000:8000 --env-file ./.env --name feedback-app feedback-node:web-app
```
Open `localhost:3000` in all cases - the browser uses the **left** (host) side of `-p`.
