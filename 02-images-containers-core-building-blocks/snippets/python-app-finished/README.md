# python-app-finished - what the Dockerfile does

```dockerfile
FROM python
WORKDIR /app
COPY . /app
CMD [ "python", "rng.py" ]
```

- `FROM python` - official Python base image (brings its own Python runtime), used instead of `node` to show the same containerization pattern works for any language.
- `WORKDIR /app` - sets the working directory inside the container.
- `COPY . /app` - copies the script into the image (no `npm install`-equivalent step needed here since this script has no external dependencies to install).
- `CMD [ "python", "rng.py" ]` - runs the script when the container starts.

Run it:
```bash
docker build -t python-app .
docker run python-app
```
