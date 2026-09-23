# Azure Container Registry (ACR) Guide

Step-by-step guide covering Azure CLI login, creating an ACR instance, authenticating to it, and pushing an image.

## auth-azure

Log in to Azure before doing anything else.

```bash
az login
```

This opens a browser window for interactive sign-in. For headless/CI environments, use a service principal instead:

```bash
az login --service-principal \
  --username <appId> \
  --password <clientSecret> \
  --tenant <tenantId>
```

Select the correct subscription if your account has access to more than one:

```bash
az account list --output table
az account set --subscription "<subscription-id-or-name>"
```

## acr-create 

If you already have one, skip to next section.
Create a resource group (if you don't already have one) and the registry itself.

```bash
az group create --name <resource-group-name> --location <region>

az acr create \
  --resource-group <resource-group-name> \
  --name <registry-name> \
  --sku Basic
```

- `<registry-name>` must be globally unique and alphanumeric (no dashes/underscores).
- SKU options: `Basic`, `Standard`, `Premium` (differ in storage, throughput, and features like geo-replication).

Verify the registry was created and note its login server:

```bash
az acr show --name <registry-name> --query loginServer --output tsv
```

## acr-list

Show your ACRs.
```
az acr list -o table
```

## acr-auth

Authenticate Docker (or another OCI client) to the registry so you can push/pull images.

```bash
az acr login --name <registry-name>
```

- Uses your current `az login` session to fetch a short-lived (~3 hour) token and configures Docker to authenticate against `<registry-name>.azurecr.io`.
- Requires Docker to be running locally, since `az acr login` invokes `docker login` under the hood.

### Alternative: token-based (no Docker)

```bash
az acr login --name <registry-name> --expose-token
```

Returns an `accessToken` usable as the password with username `00000000-0000-0000-0000-000000000000`:

```bash
docker login <registry-name>.azurecr.io \
  --username 00000000-0000-0000-0000-000000000000 \
  --password <accessToken>
```

### Alternative: service principal (CI/CD)

```bash
ACR_ID=$(az acr show --name <registry-name> --query id --output tsv)

az ad sp create-for-rbac \
  --name <sp-name> \
  --scopes $ACR_ID \
  --role acrpush \
  --output json
```

Use the returned `appId`/`password` to log in:

```bash
docker login <registry-name>.azurecr.io \
  --username <appId> \
  --password <password>
```

### Alternative: admin credentials (not recommended for production)

```bash
az acr update --name <registry-name> --admin-enabled true
az acr credential show --name <registry-name>
```

Prefer Azure AD-based auth over admin credentials for better security and auditability.

## push image to acr

Tag your local image with the registry's login server, then push.

```bash
# Build (or use an existing) local image
docker build -t <image-name>:<tag> .

# Tag it for ACR
docker tag <image-name>:<tag> <registry-name>.azurecr.io/<image-name>:<tag>

# Push
docker push <registry-name>.azurecr.io/<image-name>:<tag>
```

Verify the image landed in the registry:

```bash
az acr repository list --name <registry-name> --output table
az acr repository show-tags --name <registry-name> --repository <image-name> --output table
```

Pull it back down to confirm:

```bash
docker pull <registry-name>.azurecr.io/<image-name>:<tag>
```

## Common issues

| Issue | Fix |
|---|---|
| `az acr login` fails with "docker not found" | Ensure Docker daemon is installed and running |
| `unauthorized: authentication required` on push/pull | Re-run `az acr login --name <registry-name>`; tokens expire after ~3 hours |
| Access denied for service principal | Confirm role assignment (`AcrPull`/`AcrPush`/`AcrDelete`) via `az role assignment list --scope $ACR_ID` |
| Wrong subscription/tenant | Run `az account show` to confirm active subscription/tenant matches the ACR's |
| `denied: requested access to the resource is denied` on push | Confirm you're logged into the correct registry and have push permissions |

## References

- [az acr login docs](https://learn.microsoft.com/cli/azure/acr#az-acr-login)
- [Authenticate with Azure Container Registry](https://learn.microsoft.com/azure/container-registry/container-registry-authentication)
- [Push and pull images with az acr](https://learn.microsoft.com/azure/container-registry/container-registry-get-started-docker-cli)
