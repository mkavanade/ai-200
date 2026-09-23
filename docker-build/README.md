# Docker Build Basics

This folder contains example Dockerfiles (`bad` / `ok` / `good`) for `dotnet` and `go`
projects, showing how build structure affects image size, build speed, and caching.

## Build Context

The **build context** is the set of files sent to the Docker daemon when you run
`docker build <path>`. Everything in that path (unless excluded via `.dockerignore`)
is packed up and made available to `COPY`/`ADD` instructions.

- A larger context takes longer to send and can accidentally leak files (e.g. `.git`,
  `node_modules`, secrets) into the build.
- Use a `.dockerignore` file to exclude files/folders that aren't needed for the build.
- Only paths within the context can be used in `COPY`/`ADD` — you can't reference
  files outside of it.

## .dockerignore

A `.dockerignore` file, placed next to the Dockerfile (or in the build context root),
lists files/folders to exclude from the build context before it's sent to the Docker
daemon — similar in syntax to `.gitignore`.

Why it matters:

- **Smaller, faster builds** — less data to send to the daemon.
- **Better cache usage** — irrelevant files (e.g. editor temp files) can't invalidate
  `COPY . .` layers if they're excluded.
- **Avoids leaking sensitive/unnecessary files** into the image, such as `.git`,
  `.env`, local secrets, or `node_modules`/build output that should be rebuilt fresh.

Example `.dockerignore`:

```
.git
.gitignore
README.md
Dockerfile
.dockerignore
node_modules
bin/
obj/
*.log
.env
```

Note: `.dockerignore` only affects what's sent as build context — it does **not**
prevent already-copied files from ending up in image layers, so combine it with
careful `COPY` instructions.

## Layers

Every instruction in a Dockerfile that changes the filesystem (`RUN`, `COPY`, `ADD`)
creates a new, cached **layer**. Instructions that only change metadata (`ENV`, `CMD`,
`EXPOSE`, `LABEL`, etc.) do not create filesystem layers.

- Layers are cached and reused across builds if the instruction and its inputs
  (e.g. copied files) haven't changed.
- Once a layer's cache is invalidated, **every layer after it** must be rebuilt too —
  so order matters.
- Best practice: put things that change rarely (installing dependencies) *before*
  things that change often (copying source code), so dependency layers stay cached.

**Common beginner mistake:** copying the entire project (including source code)
before installing dependencies, e.g.:

```dockerfile
COPY . .
RUN npm install   # or: dotnet restore / go mod download
```

Any source code change invalidates the cache and forces a full dependency
reinstall. Instead, copy only the dependency manifest first:

```dockerfile
COPY package.json package-lock.json ./
RUN npm install
COPY . .
```

## Multi-Stage Builds

Multi-stage builds let you use multiple `FROM` statements in one Dockerfile — each
one starts a new stage. You can build/compile in an early stage (with full SDKs/
build tools) and copy only the compiled artifacts into a slim final stage.

- Reduces final image size significantly (no compilers, build caches, source code).
- Improves security by shrinking the attack surface of the runtime image.
- Example pattern: `build` stage uses `golang`/`dotnet-sdk` image, `final` stage
  uses `alpine`/`aspnet` runtime image, copying only the built binary/DLLs across.

## Best Practices

- **Use small base images.** Prefer `alpine`, `distroless`, or slim/runtime variants
  over full SDK/OS images for the final stage.
- **Use multi-stage builds** to keep build tools and source code out of the final image.
- **Pin base image versions** (e.g. `golang:1.22-alpine`, not `golang:latest`) for
  reproducible builds.
- **Order instructions from least to most frequently changing** so Docker's layer
  cache is reused as much as possible (deps before source code).
- **Combine related `RUN` commands** with `&&` to avoid unnecessary layers, and clean
  up in the same layer (e.g. `apt-get install ... && rm -rf /var/lib/apt/lists/*`)
  so temp files don't bloat the image.
- **Use a `.dockerignore`** to keep the build context small and avoid leaking secrets,
  `.git`, or local artifacts into the image.
- **Don't run as root.** Create and switch to a non-root user (`USER appuser`) in
  the final stage.
- **Avoid baking secrets into layers.** Use build secrets (`--mount=type=secret`) or
  runtime environment variables/secret stores instead of `ARG`/`ENV` for credentials.
- **Use explicit `COPY` instead of `ADD`** unless you specifically need `ADD`'s
  tar-extraction or remote-URL behavior.
- **Set `CMD`/`ENTRYPOINT` explicitly** and prefer the exec form (`["binary", "arg"]`)
  over the shell form for proper signal handling (e.g. graceful shutdown on SIGTERM).
- **Tag images meaningfully** (git SHA, semver) rather than relying only on `latest`.
- **Scan images** for vulnerabilities (e.g. `docker scout`, `trivy`) as part of CI.

## bad / ok / good

The subfolders in `dotnet/` and `go/` illustrate this progression:

- **bad** — single-stage build, copies everything up front, poor caching, bloated image.
- **ok** — separates dependency install from source copy for better caching, still single-stage.
- **good** — multi-stage build with proper layer ordering and a minimal final image.

[Read more](https://labs.iximiuz.com/tutorials/docker-multi-stage-builds)