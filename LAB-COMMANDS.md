# Azure Voting App Lab --- CLI Command Reference

This file collects the commands used or discussed during the lab, plus
useful diagnostic commands for repeating it. Commands marked
**example/template** need values adjusted to your subscription,
repository, or image namespace. Portal-only steps and workflow YAML are
described separately.

> **Important:** Azure resource creation/deletion commands can create
> charges or permanently delete resources. Review names and subscription
> before running them. Do not paste tokens or secrets into shell
> history.

https://docs.github.com/en/rest/orgs/orgs?apiVersion=2026-03-10
https://docs.github.com/en/rest/repos/repos?apiVersion=2026-03-10

## 1. Set up the shell and Azure subscription

```bash
az --version
az login
az account list --output table
az account show --output table
az account set --subscription "<SUBSCRIPTION_ID>"
az account show --query "{subscription:name,id:id,tenantId:tenantId}" --output table
```

## 2. Create the resource group and AKS cluster

The lab values were: - Resource group: `rg-azure-voting-lab` - Cluster:
`aks-voting-lab` - Region: `centralindia` - Node count: `1` - VM size
used in the final lab: `Standard_D2as_v4`

```bash
az group create \
  --name rg-azure-voting-lab \
  --location centralindia

az aks create \
  --resource-group rg-azure-voting-lab \
  --name aks-voting-lab \
  --location centralindia \
  --node-count 1 \
  --node-vm-size Standard_D2as_v4 \
  --generate-ssh-keys \
  --enable-aad \
  --enable-azure-rbac
```

> AKS creation flags and available VM sizes can change. If recreating
> the cluster, check the current AKS CLI documentation and region quota.
> If your tenant requires a specific Entra/RBAC configuration, follow
> the current Azure portal or CLI flow rather than blindly rerunning the
> command.

## 3. Inspect AKS and configure kubectl

```bash
az aks show \
  --resource-group rg-azure-voting-lab \
  --name aks-voting-lab \
  --output table

az aks get-credentials \
  --resource-group rg-azure-voting-lab \
  --name aks-voting-lab \
  --overwrite-existing

az aks install-cli
kubelogin convert-kubeconfig -l azurecli

kubectl version --client
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
```

If credentials already exist for a different cluster,
`--overwrite-existing` replaces the local context entry for the same
cluster name.

## 4. Azure RBAC and role-assignment checks

The lab used Microsoft Entra ID authentication with Azure RBAC. The
GitHub Actions identity was assigned: - Azure Kubernetes Service Cluster
User Role - Azure Kubernetes Service RBAC Writer

The operator was assigned Azure Kubernetes Service RBAC Reader to view
workloads in the Azure portal.

```bash
az aks show \
  --resource-group rg-azure-voting-lab \
  --name aks-voting-lab \
  --query "{name:name,enableAzureRBAC:enableAzureRBAC,azureAD: aadProfile.managed}" \
  --output json

AKS_ID=$(az aks show \
  --resource-group rg-azure-voting-lab \
  --name aks-voting-lab \
  --query id --output tsv)

az role assignment list \
  --scope "$AKS_ID" \
  --all \
  --output table
```

**Example/template** for role assignments (requires the appropriate
Azure permissions). Replace `<PRINCIPAL_OBJECT_ID>` with the service
principal's object ID, not its client ID:

```bash
az role assignment create \
  --assignee-object-id "<PRINCIPAL_OBJECT_ID>" \
  --assignee-principal-type ServicePrincipal \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope "$AKS_ID"

az role assignment create \
  --assignee-object-id "<PRINCIPAL_OBJECT_ID>" \
  --assignee-principal-type ServicePrincipal \
  --role "Azure Kubernetes Service RBAC Writer" \
  --scope "$AKS_ID"
```

To find the service principal object ID by application/client ID:

```bash
az ad sp show --id "<AZURE_CLIENT_ID>" --query id --output tsv
```

The OIDC app registration and federated credential were configured for
the GitHub repository and `main` branch. Check current Azure CLI syntax
before scripting identity/federated-credential creation; this lab also
used the Azure portal for parts of that configuration.

## 5. Docker commands for local practice

```bash
docker --version
docker info
docker build -t azure-vote-front:local ./azure-vote
docker build -t azure-vote-redis:local ./redis
docker images
docker login
```

**Example/template** to tag and publish to Docker Hub. Use your own
namespace and a unique tag; the CI workflow uses the Git commit SHA.

```bash
export DOCKERHUB_USERNAME="<DOCKERHUB_USERNAME>"
export IMAGE_TAG="<COMMIT_SHA>"

docker tag azure-vote-front:local \
  "${DOCKERHUB_USERNAME}/azure-vote-front:${IMAGE_TAG}"

docker tag azure-vote-redis:local \
  "${DOCKERHUB_USERNAME}/azure-vote-redis:${IMAGE_TAG}"

docker push "${DOCKERHUB_USERNAME}/azure-vote-front:${IMAGE_TAG}"
docker push "${DOCKERHUB_USERNAME}/azure-vote-redis:${IMAGE_TAG}"
```

Use `docker login` interactively or with a credential manager. Avoid
putting access tokens directly in a command that may be saved in shell
history.

## 6. Git commands used for the repository

```bash
git clone https://github.com/vijeshcloudops/azure-voting-app-redis.git
cd azure-voting-app-redis
git status
git branch
git pull origin main
git add README.md LAB-COMMANDS.md
git commit -m "Document Azure Voting App CI/CD lab"
git push origin main
```

Useful checks for the manifest and workflows:

```bash
git diff
git diff -- deployment/aks/azure-vote.yaml
git log --oneline --decorate -10
git show --stat --oneline HEAD
```

## 7. Kubernetes deploy and verification commands

```bash
kubectl apply -f deployment/aks/azure-vote.yaml
kubectl get deployments
kubectl get pods -o wide
kubectl get services
kubectl get all
kubectl get events --sort-by=.metadata.creationTimestamp
```

Wait for the deployments to finish:

```bash
kubectl rollout status deployment/azure-vote-front --timeout=180s
kubectl rollout status deployment/azure-vote-back --timeout=180s
```

Check which image tags are actually configured on the running
deployments:

```bash
kubectl get deployment azure-vote-front \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'

kubectl get deployment azure-vote-back \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

Get the frontend external IP:

```bash
kubectl get service azure-vote-front
kubectl get service azure-vote-front \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```

## 8. Troubleshoot Kubernetes workloads

```bash
kubectl describe deployment azure-vote-front
kubectl describe deployment azure-vote-back
kubectl describe pods
kubectl describe service azure-vote-front
kubectl logs deployment/azure-vote-front
kubectl logs deployment/azure-vote-back
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get pods -o wide
kubectl get endpoints azure-vote-back
```

For a specific pod:

```bash
kubectl logs "<POD_NAME>"
kubectl logs "<POD_NAME>" --previous
kubectl describe pod "<POD_NAME>"
```

Check rollout history and roll back only when you understand which
revision is known-good:

```bash
kubectl rollout history deployment/azure-vote-front
kubectl rollout history deployment/azure-vote-back
# Optional recovery:
kubectl rollout undo deployment/azure-vote-front
kubectl rollout undo deployment/azure-vote-back
```

If a pod reports `ImagePullBackOff` or `ErrImagePull`, verify: 1. The
image reference and tag in `deployment/aks/azure-vote.yaml`. 2. That the
image tag exists in Docker Hub. 3. That the repository is public or the
cluster has valid image-pull credentials if it is private.

If the frontend Service has no external IP yet, check:

```bash
kubectl describe service azure-vote-front
kubectl get events --sort-by=.metadata.creationTimestamp
```

## 9. Manifest image updates with sed

CI updates the manifest using `sed`, validates the resulting image
lines, then commits and pushes the change. This command changes a local
file; it becomes a permanent repository change only after `git commit`
and `git push`.

**Example/template**:

```bash
export DOCKERHUB_USERNAME="<DOCKERHUB_USERNAME>"
export IMAGE_TAG="<COMMIT_SHA>"
MANIFEST="deployment/aks/azure-vote.yaml"

sed -i \
  "s|^\([[:space:]]*image: \).*azure-vote-front:.*|\1${DOCKERHUB_USERNAME}/azure-vote-front:${IMAGE_TAG}|" \
  "$MANIFEST"

sed -i \
  "s|^\([[:space:]]*image: \).*azure-vote-redis:.*|\1${DOCKERHUB_USERNAME}/azure-vote-redis:${IMAGE_TAG}|" \
  "$MANIFEST"

grep -E 'image: .*azure-vote-(front|redis):' "$MANIFEST"
git diff -- "$MANIFEST"
```

The CI workflow uses `GITHUB_SHA` as `IMAGE_TAG` and only updates the
manifest after the image builds, scans, and publishes succeed.

## 10. Azure resource troubleshooting

```bash
az group show --name rg-azure-voting-lab --output table

az aks show \
  --resource-group rg-azure-voting-lab \
  --name aks-voting-lab \
  --output table

az aks nodepool list \
  --resource-group rg-azure-voting-lab \
  --cluster-name aks-voting-lab \
  --output table

az monitor activity-log list \
  --resource-group rg-azure-voting-lab \
  --max-events 30 \
  --output table
```

Check whether the resource group still exists:

```bash
az group exists --name rg-azure-voting-lab
```

For subscription cost review, use **Azure Portal → Cost Management +
Billing → Cost analysis**. Billing data can lag behind resource
deletion.

## 11. Tear down the lab

First inspect the resource group and confirm it contains only lab
resources you intend to delete:

```bash
az group show --name rg-azure-voting-lab --output table
az resource list --resource-group rg-azure-voting-lab --output table
```

Delete the complete lab resource group:

```bash
az group delete \
  --name rg-azure-voting-lab \
  --yes
```

Verify deletion:

```bash
az group exists --name rg-azure-voting-lab
```

Expected output after deletion:

```text
false
```

Deleting this resource group does not delete GitHub repository files,
GitHub Actions secrets, or Docker Hub images. Check for resources
created outside the group separately.

## 12. GitHub Actions workflow checks

Use the GitHub web interface: - **Repository → Actions** to inspect CI
and CD runs. - Open the failed job and inspect the exact failing step
and logs. - Confirm the repository secrets exist and are spelled
correctly. - Confirm the CI workflow is named `Azure Voting App - CI`,
matching any workflow-name trigger that references it. - Confirm the CD
workflow path filter includes `deployment/aks/azure-vote.yaml`. -
Confirm `GH_PAT` has repository Contents read/write access if the CI
manifest commit fails.

The workflows use: - `azure/login@v2` for Azure OIDC login. -
`az aks install-cli` to install `kubectl` and `kubelogin`. -
`kubelogin convert-kubeconfig -l azurecli` for Entra-backed Kubernetes
authentication. - `kubectl apply` to apply the committed manifest. -
`kubectl rollout status` to verify the frontend and Redis deployments.

## Notes on command history

This document consolidates the commands recorded during the lab and adds
standard diagnostic examples that are useful when repeating it. It is
not a verbatim terminal transcript. Azure portal actions (such as
creating the federated credential and changing AKS authentication
settings) do not necessarily have an exact CLI command recorded here.
Always verify current Azure CLI syntax and subscription settings before
recreating infrastructure.
