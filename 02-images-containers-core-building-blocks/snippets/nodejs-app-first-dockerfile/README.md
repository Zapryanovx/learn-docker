# nodejs-app-first-dockerfile - what the Dockerfile does

```dockerfile
FROM node
WORKDIR /app
COPY . /app
RUN npm install
EXPOSE 80
CMD ["node", "server.js"]
```

This is the **naive, unoptimized** version: everything is copied (`COPY . /app`) *before* `npm install` runs. That means any change to any file, even just `server.js`, invalidates the cache for this layer and forces `npm install` to rerun on every single rebuild, even though the dependencies themselves haven't changed. Compare with `../nodejs-app-finished/`, where `package.json` is copied separately first specifically to avoid this.

Run it:
```bash
docker build -t nodejs-app .
docker run -p 3000:80 nodejs-app
```
