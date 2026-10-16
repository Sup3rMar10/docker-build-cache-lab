# docker-build-cache-lab

> **Reference only.** The two Dockerfiles were written to show a point. Both pass `docker build --check` (Docker's Dockerfile linter, no warnings), but neither image was built and the timing experiment below was **not run**.

The same tiny Python app built two ways. The difference is layer order, and it decides whether a one-line code change costs seconds or minutes.

| File | Order | Effect of editing `main.py` |
|---|---|---|
| `Dockerfile.bad` | copy everything, then `pip install` | `pip install` runs again |
| `Dockerfile.good` | copy `requirements.txt`, install, then copy source | install layer comes from cache |

## The rule

Docker caches a layer until its inputs change, and **every layer after a changed one is rebuilt**. Order instructions from least to most frequently changing.

## Also here

- **`.dockerignore`** keeps `.git`, caches and virtualenvs out of the build context. A smaller context uploads faster and stops irrelevant file changes from busting `COPY` caches.
- **Cache mount** (`RUN --mount=type=cache`): even when the install layer must re-run, pip reuses downloaded wheels. Needs BuildKit (default in current Docker).
- **Non-root user** in the good file.

## Try it yourself

```
docker build -f Dockerfile.bad  -t lab:bad  .
docker build -f Dockerfile.good -t lab:good .
echo "# edit" >> app/main.py          # change source only
docker build -f Dockerfile.bad  -t lab:bad  .    # pip install runs again
docker build -f Dockerfile.good -t lab:good .    # install step says CACHED
```

Watch for `CACHED` next to the install step. Compare wall-clock times; do it a few times, the first run includes pulls.

## What was checked

`docker build --check` on both files (2026-10-16): no warnings. Nothing about build times or cache hits is claimed.

## Not done

No measurements, no multi-stage build, no vulnerability scan, no pinned base-image digest. `flask==3.0.3` is a placeholder dependency; the app does not use it.

MIT licensed.
