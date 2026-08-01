# first-demo - what the Dockerfile does

```dockerfile
FROM node
WORKDIR /app
COPY package.json /app
RUN npm install
COPY . /app
EXPOSE 3000
CMD ["node", "app.mjs"]
```

- `FROM node` - start from the official Node.js image (brings its own Node.js runtime).
- `WORKDIR /app` - all following instructions run inside `/app` in the container.
- `COPY package.json /app` - copy just the dependency list first.
- `RUN npm install` - install dependencies (this layer gets cached, only reruns if `package.json` changes).
- `COPY . /app` - copy the rest of the source code.
- `EXPOSE 3000` - documents that the app listens on port 3000 (no real effect on its own, needs `-p` at `docker run`).
- `CMD ["node", "app.mjs"]` - command that runs when the container starts.

Run it:
```bash
docker build -t first-demo .
docker run -p 3000:3000 first-demo
```
