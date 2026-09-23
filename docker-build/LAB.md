# Docker Build Lab

Hands-on lab: build the `bad` / `ok` / `good` Dockerfiles in this folder, push them to
Azure Container Registry (ACR), and configure a cleanup rule that deletes images older
than 6 days.

Prerequisites: Docker running locally, an ACR instance (see [`../acr/LAB.md`](../acr/LAB.md)
for creating one and authenticating), and `az` CLI logged in.

```bash
REGISTRY_NAME=<registry-name>            # e.g. myregistry
REGISTRY=$REGISTRY_NAME.azurecr.io
```

## build-images

Each language folder has three Dockerfiles (no extension) demonstrating the same app
built with progressively better practices. Build them with `-f` since they aren't
named `Dockerfile`, using the language folder as the build context.

Change to directory to `docker-build`...

```bash
# dotnet
docker build -f dotnet/bad  -t hello-dotnet:bad  dotnet/
docker build -f dotnet/ok   -t hello-dotnet:ok   dotnet/
docker build -f dotnet/good -t hello-dotnet:good dotnet/

# go
docker build -f go/bad  -t hello-go:bad  go/
docker build -f go/ok   -t hello-go:ok   go/
docker build -f go/good -t hello-go:good go/
```

Compare image sizes to see the impact of multi-stage builds and smaller base images:

```bash
docker images | grep -E "hello-dotnet|hello-go"
```

Sanity check a build runs:

```bash
docker run --rm -p 8080:8080 hello-dotnet:good &
curl localhost:8080
docker run --rm -p 8090:8090 hello-go:good &
curl localhost:8090/hello
```

## tag-images

Tag each local image for the registry. Include a version/date tag alongside `latest`
so you have multiple tags to observe the cleanup rule acting on later.

```bash
for img in hello-dotnet:bad hello-dotnet:ok hello-dotnet:good hello-go:bad hello-go:ok hello-go:good; do
  docker tag "$img" "$REGISTRY/$img"
done
```

## push-images

Authenticate to ACR, then push.

```bash
az acr login --name $REGISTRY_NAME

for img in hello-dotnet:bad hello-dotnet:ok hello-dotnet:good hello-go:bad hello-go:ok hello-go:good; do
  docker push "$REGISTRY/$img"
done
```

Verify:

```bash
az acr repository list --name $REGISTRY_NAME --output table
az acr repository show-tags --name $REGISTRY_NAME --repository hello-dotnet --output table
az acr repository show-tags --name $REGISTRY_NAME --repository hello-go --output table
```

## cleanup-rule

Goal: automatically delete images older than **6 days** so the registry doesn't grow
unbounded with lab/test images. ACR doesn't have a simple "max age" toggle in the
portal for tagged images — the supported approaches are below.

### Option A: ACR Tasks + `acr-cli purge` (scheduled, tag-aware)

The `acr-cli` `purge` command deletes tagged/untagged manifests older than a given
duration, with optional filters. Run it on a schedule via `az acr task create`.

```bash
az acr task create \
  --registry $REGISTRY_NAME \
  --name purge-old-images \
  --cmd "mcr.microsoft.com/acr/acr-cli:latest purge --filter 'hello-dotnet:.*' --filter 'hello-go:.*' --ago 6d --untagged" \
  --schedule "0 2 * * *" \
  --context /dev/null
```

- `--ago 6d` — deletes images last updated more than 6 days ago.
- `--filter '<repository>:<tag-regex>'` — required per repository; `.*` matches all tags.
- `--untagged` — also removes dangling manifests with no tags.
- `--schedule "0 2 * * *"` — cron expression; runs daily at 02:00 UTC.
- Add `--dry-run` first to preview what would be deleted before enabling it for real.

Trigger a manual run to test immediately instead of waiting for the schedule:

```bash
az acr task run --registry $REGISTRY_NAME --name purge-old-images
```

Check run history/logs:

```bash
az acr task list-runs --registry $REGISTRY_NAME --output table
az acr task logs --registry $REGISTRY_NAME --run-id <run-id>
```

### Option B: Retention policy for untagged manifests only (built-in, Premium SKU)

ACR has a native **retention policy**, but it only applies to **untagged** manifests
(e.g. left behind after re-pushing a tag) — it does not expire tagged images by age.
Useful as a complement to Option A, not a replacement.

```bash
az acr config retention update \
  --registry $REGISTRY_NAME \
  --status enabled \
  --days 6 \
  --type UntaggedManifests
```

```bash
az acr config retention show --registry $REGISTRY_NAME
```

### Which to use

| Requirement | Option A (ACR Tasks + purge) | Option B (retention policy) |
|---|---|---|
| Deletes tagged images by age | Yes | No |
| Deletes untagged/dangling manifests | Yes (`--untagged`) | Yes |
| Requires scheduling/cron | Yes | No (always-on) |
| SKU requirement | Any | Premium only |

For "delete anything older than 6 days" as requested, **Option A** is the one that
actually satisfies the requirement; use Option B alongside it to also clean up
untagged manifest debris.

## verify-cleanup

After the purge task runs (manually or on schedule), confirm old tags are gone:

```bash
az acr repository show-tags --name $REGISTRY_NAME --repository hello-dotnet --orderby time_asc --output table
az acr repository show-tags --name $REGISTRY_NAME --repository hello-go --orderby time_asc --output table
```

## cleanup-lab-resources

Remove local test images and, if this was a throwaway lab, the purge task itself.

```bash
docker rmi $(docker images "hello-dotnet" -q) $(docker images "hello-go" -q) 2>/dev/null

az acr task delete --registry $REGISTRY_NAME --name purge-old-images --yes
```

## References

- [ACR Tasks: purge repositories](https://learn.microsoft.com/azure/container-registry/container-registry-auto-purge)
- [acr-cli purge command reference](https://github.com/Azure/acr-cli)
- [Retention policy for untagged manifests](https://learn.microsoft.com/azure/container-registry/container-registry-retention-policy)
- [`../acr/LAB.md`](../acr/LAB.md) — registry creation and authentication
