# nodejs-app-finished - what the Dockerfile does

```dockerfile
FROM node:12
WORKDIR /app
COPY package.json /app
RUN npm install
COPY . /app
EXPOSE 80
CMD ["node", "server.js"]
```

Optimized layer order (dependencies copied and installed before the rest of the code, for better build caching, see `../nodejs-app-first-dockerfile/` for the naive alternative this fixes). It also pins an exact version of the base image, `FROM node:12`, instead of using `node` (which would silently pull whatever the current "latest" tag is). Pinning avoids surprises where a rebuild months later grabs a newer Node version and behaves differently.

Run it:
```bash
docker build -t nodejs-app .
docker run -p 3000:80 nodejs-app
```
