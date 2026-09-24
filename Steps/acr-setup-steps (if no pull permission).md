# AKS + ACR Setup — Repeatable Steps

## Goal

Configure an Azure Container Registry (ACR) and use its images from an Azure Kubernetes Service (AKS) cluster.

Example names used:
- AKS: `kube`
- ACR: `ndacr`

## 1. Create the ACR

Create the Azure Container Registry `ndacr`.

After creation, verify it in Azure Portal under:
**Container Registries → ndacr**

For this setup, ACR uses:
**Role assignment permissions mode: RBAC Registry Permissions**

## 2. Authenticate to Azure

From the local machine:

```bash
az login
```

Verify the active Azure subscription if needed:

```bash
az account show
```

## 3. Make sure Docker Desktop is running

`az acr login` uses the local Docker CLI/daemon.

Verify Docker:

```bash
docker version
```

Start Docker Desktop if it is not running.

## 4. Log in to ACR

```bash
az acr login --name ndacr
```

Expected result:

```text
Login Succeeded
```

## 5. Build the application image

From the application directory containing the `Dockerfile`:

```bash
docker build -t myapp:v1.0.0 .
```

The `.` is the Docker build context (the current application directory).

Verify:

```bash
docker images
```

## 6. Tag the image for ACR

```bash
docker tag myapp:v1.0.0 ndacr.azurecr.io/myapp:v1.0.0
```

ACR repository naming convention:

```text
ndacr.azurecr.io/<repository>:<tag>
```

For multiple applications, use separate repositories, for example:

```text
ndacr.azurecr.io/frontend:v1.0.0
ndacr.azurecr.io/backend:v1.0.0
ndacr.azurecr.io/api:v1.0.0
```

## 7. Push the image to ACR

```bash
docker push ndacr.azurecr.io/myapp:v1.0.0
```

ACR creates the repository automatically when the first image is pushed.

Verify repositories:

```bash
az acr repository list --name ndacr -o table
```

Verify tags:

```bash
az acr repository show-tags \
  --name ndacr \
  --repository myapp \
  -o table
```

## 8. Create the ACR repository scope map

Use this when AKS cannot be granted `AcrPull` through Azure RBAC.

### Portal

Go to:

**Azure Portal → Container Registries → ndacr → Permissions → Scope maps → Create**

Set:

```text
Name: aks-pull-scope
```

Add repository permission:

```text
Repository: myapp
Actions: content/read
```

For multiple applications, create the scope map with each required repository:

```text
frontend  → content/read
backend   → content/read
api       → content/read
```

Do not grant:

```text
content/write
content/delete
```

### Automation

```bash
az acr scope-map create   --registry ndacr   --name aks-pull-scope   --repository myapp content/read
```

For multiple repositories:

```bash
az acr scope-map create   --registry ndacr   --name aks-pull-scope   --repository frontend content/read   --repository backend content/read   --repository api content/read
```

Verify:

```bash
az acr scope-map show   --registry ndacr   --name aks-pull-scope
```

## 9. Create the ACR token

### Portal

Go to:

**Azure Portal → Container Registries → ndacr → Permissions → Tokens → Create**

Set:

```text
Token name: aks-pull-token
Scope map: aks-pull-scope
```

Create the token.

Then open the token and generate/reset its passwords.

Record securely:

```text
Username = aks-pull-token
Password = <generated-token-password>
```

Use only the token password required for authentication.

### Automation

Create the token:

```bash
az acr token create   --registry ndacr   --name aks-pull-token   --scope-map aks-pull-scope
```

Generate a password:

```bash
az acr token credential generate   --registry ndacr   --name aks-pull-token   --password-name password1
```

Retrieve the credential when needed:

```bash
az acr token credential show   --registry ndacr   --name aks-pull-token
```

Keep the token password secret. Do not commit it to Git.

## 10. Create the Kubernetes image-pull Secret

### Declarative YAML

Create `acr-pull-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: acr-pull-secret
  namespace: <your-namespace>
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "ndacr.azurecr.io": {
          "username": "<ACR_TOKEN_NAME>",
          "password": "<ACR_TOKEN_PASSWORD>"
        }
      }
    }
```

Apply:

```bash
kubectl apply -f acr-pull-secret.yaml
```

Verify:

```bash
kubectl get secret acr-pull-secret -n <your-namespace>
```

### Imperative

```bash
kubectl create secret docker-registry acr-pull-secret   --namespace <your-namespace>   --docker-server=ndacr.azurecr.io   --docker-username=aks-pull-token   --docker-password='<ACR_TOKEN_PASSWORD>'
```

Verify:

```bash
kubectl get secret acr-pull-secret -n <your-namespace>
```

Expected type:

```text
kubernetes.io/dockerconfigjson
```

Do not commit the YAML containing the actual token password to Git.

## 11. Reference the Secret from the Deployment

The Deployment must reference the image-pull Secret in the same namespace:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: acr-pull-secret
      containers:
        - name: myapp
          image: ndacr.azurecr.io/myapp:v1.0.0
```

### Portal

Go to:

**Azure Portal → Kubernetes services → kube → Workloads → Deployments → <deployment> → Edit**

Set the container image:

```text
ndacr.azurecr.io/myapp:v1.0.0
```

Add the image-pull Secret:

```text
acr-pull-secret
```

Save.

### Declarative

Update the Deployment YAML:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: acr-pull-secret
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

## 12. Verify the workload

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: acr-pull-secret
  namespace: <your-namespace>
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "ndacr.azurecr.io": {
          "username": "<ACR_TOKEN_NAME>",
          "password": "<ACR_TOKEN_PASSWORD>"
        }
      }
    }
```

`auth` does not need to be manually supplied when username/password are provided.

Apply it:

```bash
kubectl apply -f acr-pull-secret.yaml
```

Verify:

```bash
kubectl get secret acr-pull-secret -n <your-namespace>
```

Expected type:

```text
kubernetes.io/dockerconfigjson
```

## 10. Reference the Secret from the Deployment

The Deployment must reference the image-pull Secret in the same namespace:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: acr-pull-secret
      containers:
        - name: myapp
          image: ndacr.azurecr.io/myapp:v1.0.0
```

The image-pull Secret is namespace-scoped, so the Secret and Pod must be in the same namespace.

## 11. Verify the workload

```bash
kubectl get pods -n <your-namespace>
```

If image pulling fails:

```bash
kubectl describe pod <pod-name> -n <your-namespace>
```

Check the Pod Events for image-pull/authentication errors.

## 13. Optional: test the ACR credential before deploying

Authenticate with the ACR token:

```bash
docker login ndacr.azurecr.io   --username aks-pull-token   --password <ACR_TOKEN_PASSWORD>
```

Pull the image:

```bash
docker pull ndacr.azurecr.io/myapp:v1.0.0
```

Log out:

```bash
docker logout ndacr.azurecr.io
```

## Enterprise notes

- Preferred native AKS integration: AKS kubelet managed identity with ACR pull permission (`AcrPull` for RBAC Registry Permissions).
- That native approach requires permission to create the Azure RBAC role assignment.
- If that permission is unavailable, an ACR repository token + Kubernetes `imagePullSecret` is the workable alternative.
- Keep the ACR token out of Git/source control. Use a secret-management solution for production.
- AKS should have pull/read access only; CI/CD identities should be responsible for image push/write.
- Use separate ACR repositories for separate applications.
- Prefer immutable image versions/tags rather than relying on `latest`.
