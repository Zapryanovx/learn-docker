# favorites-app - what the Dockerfile does

```dockerfile
FROM node
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
CMD ["node", "app.js"]
```

Same cached layer order as before. No `EXPOSE` - the app listens on 3000, published with `-p` on `docker run`. `.dockerignore` keeps a local `node_modules` (from running the app without Docker) out of `COPY . .`.

The app is a small web API (Express + Axios + Mongoose):

| Endpoint | Talks to | Scenario |
|---|---|---|
| `GET /movies`, `GET /people` | Star Wars API (`swapi.dev`) | container → internet |
| `POST /favorites`, `GET /favorites` | MongoDB | container → host, later container → container |

`POST /favorites` body (JSON): `{ "name": "A New Hope", "type": "movie", "url": "https://swapi.dev/api/films/1/" }` - `type` must be `movie` or `character`.

Note: `app.listen(3000)` is called **inside** the `mongoose.connect` callback, so if the database is unreachable the server never starts and every endpoint (even `/movies`) gives `ECONNREFUSED`.

## The connection string through the module

The only thing that changes is the host part of the MongoDB URL in `app.js` (rebuild the image after every change - the code is baked in):

| Where MongoDB runs | URL in `app.js` |
|---|---|
| installed on the host, app run with `node app.js` | `mongodb://localhost:27017/swfavorites` |
| on the host (or in a container with `-p 27017:27017`), app in a container | `mongodb://host.docker.internal:27017/swfavorites` |
| in a container, found by IP (bad: IP changes) | `mongodb://172.17.0.2:27017/swfavorites` |
| in a container on the same network (final version) | `mongodb://mongodb:27017/swfavorites` |

## Run the final version

```bash
docker network create favorites-net
docker run -d --name mongodb --network favorites-net mongo
docker build -t favorites-node .
docker run -d --rm --name favorites --network favorites-net -p 3000:3000 favorites-node
```

- `mongodb` needs no `-p`: only the API container talks to it, inside the network.
- `favorites` needs `-p 3000:3000`: Postman/browser on the host talk to it.
- Test with Postman: `GET localhost:3000/movies`, `POST` / `GET localhost:3000/favorites`.

The database data lives in the `mongodb` container layer, so it's lost when that container is removed - fixed with a volume in the next module.
