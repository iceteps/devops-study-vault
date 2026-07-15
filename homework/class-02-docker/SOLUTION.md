# Solution — Docker Basics Assignment 1

- **Docker Hub repo:** https://hub.docker.com/r/iceteps/hello-docker-app (public, tags `1.0` and `latest`)
- **Project folder:** `C:\Users\icete\github\Devops\docker-basics-assignment1` (not in this vault repo —
  stdlib-only `app.py` per the assignment spec, hand-written `Dockerfile`)
- **Status:** done 2026-07-15 — built, ran, verified on port 8080, pushed, confirmed public via the
  Docker Hub API.

```dockerfile
# official Python base image
FROM python:3.11-slim

# working directory inside the image
WORKDIR /app

# copy the application code into the image
COPY app.py .

# document the port the app listens on
EXPOSE 8080

# start the application when the container runs
CMD ["python", "app.py"]
```

**FROM** picks the base image everything else layers on top of — here, official slim Python 3.11.
**COPY** puts `app.py` from the build context into the image filesystem at `WORKDIR`.
**CMD** is the default process the container runs on `docker run` (overridable at the CLI).
**Why login is required for a public repo:** *public* only controls *read* access (anyone can `pull`).
*Writing* to a namespace you don't own would let anyone overwrite anyone else's images, so `push`
always authenticates — Docker Hub has to verify you actually own that repository first.
