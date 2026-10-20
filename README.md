---
description: "Hands-on Azure DevOps lab: build, scan, publish, and
  deploy a Python/Flask and Redis application to AKS using GitHub
  Actions, Docker Hub, Trivy, and Azure OIDC."
languages:
  - python
page_type: sample
products:
  - azure
  - azure-kubernetes-service
---

# Azure Voting App --- Azure DevOps CI/CD Lab

A hands-on lab for learning how to build a CI/CD pipeline for a
containerized, multi-container application and deploy it to Azure
Kubernetes Service (AKS).

The application has a Python/Flask frontend and a Redis backend. GitHub
Actions builds and scans both container images, publishes versioned
images to Docker Hub, updates the Kubernetes manifest in Git, and
triggers a separate deployment workflow to AKS.

> **Lab status:** The pipeline was tested successfully. The Azure lab
> resources have since been torn down to avoid ongoing charges. Follow
> the setup instructions to recreate the environment.

## What you will learn

- Build Docker images for a Python application and Redis.
- Use GitHub Actions for CI validation and automation.
- Scan container images with Trivy and fail the pipeline on configured
  HIGH/CRITICAL findings.
- Publish immutable, commit-SHA-tagged images to Docker Hub.
- Authenticate GitHub Actions to Azure using OpenID Connect (OIDC),
  without storing an Azure client secret.
- Configure AKS access and deploy using `kubectl`.
- Keep Kubernetes image tags committed in Git as the deployment
  configuration.
- Verify rollouts and troubleshoot common AKS and pipeline issues.
- Clean up Azure resources after the lab to reduce cost.

## Architecture

```text
Developer pushes to main
          |
          v
GitHub Actions — CI
  1. Validate required files
  2. Build frontend and Redis images
  3. Scan both images with Trivy
  4. Push images to Docker Hub using the commit SHA
  5. Update deployment/aks/azure-vote.yaml
  6. Commit the manifest update to main
          |
          v
GitHub Actions — CD
  Triggered by a change to deployment/aks/azure-vote.yaml
  1. Authenticate to Azure using OIDC
  2. Obtain AKS credentials
  3. Apply the committed Kubernetes manifest
  4. Wait for frontend and Redis rollouts
          |
          v
Azure Kubernetes Service
  Frontend Service (LoadBalancer) --> Flask frontend --> Redis Service --> Redis pod
```

### Source of truth

The Kubernetes manifest is stored at `deployment/aks/azure-vote.yaml`.
CI updates the frontend and Redis image tags and commits that change to
GitHub. CD applies the committed manifest; it does not use
`kubectl set image`.

The CI workflow ignores pushes that change only
`deployment/aks/azure-vote.yaml`, preventing the manifest-update commit
from starting another CI build. The separate CD workflow listens for
changes to that manifest.

## Repository structure

```text
.
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy-aks.yml
├── azure-vote/
│   ├── Dockerfile
│   └── azure-vote/
│       └── main.py
├── redis/
│   └── Dockerfile
├── deployment/
│   └── aks/
│       └── azure-vote.yaml
├── README.md
└── LAB-COMMANDS.md
```

## Prerequisites

- A GitHub account and this repository.
- An Azure subscription with permission to create resource groups and
  AKS clusters and configure role assignments.
- Azure CLI.
- `kubectl` and `kubelogin` (the workflow installs them).
- Docker Desktop or Docker Engine for local image-building practice.
- A Docker Hub account.
- A GitHub fine-grained personal access token (PAT) with **Contents:
  Read and write** access to this repository, used by CI to commit the
  updated manifest.
- Basic familiarity with Git, containers, YAML, and Kubernetes.

## GitHub repository secrets

Configure these under **Repository → Settings → Secrets and variables →
Actions**.

---

Secret Purpose

---

`DOCKERHUB_USERNAME` Docker Hub username / image
namespace

`DOCKERHUB_TOKEN` Docker Hub access token used to
push images

`GH_PAT` Fine-grained GitHub PAT with
repository Contents read/write
permission, used to push the
manifest commit

`AZURE_CLIENT_ID` Client/application ID of the Entra
application or service principal
used by GitHub OIDC

`AZURE_TENANT_ID` Microsoft Entra tenant ID

`AZURE_SUBSCRIPTION_ID` Azure subscription ID

---

Do not commit secrets, PATs, client secrets, kubeconfig files, or access
tokens to Git.

## Azure setup

The tested lab used these values:

Setting Value

---

Resource group `rg-azure-voting-lab`
AKS cluster `aks-voting-lab`
Region `centralindia`
Node count `1`
Node VM size `Standard_D2as_v4`
Kubernetes authentication Microsoft Entra ID with Azure RBAC
GitHub branch `main`

The cluster was configured for Microsoft Entra authentication and Azure
RBAC. GitHub Actions used an OIDC federated credential scoped to this
repository and branch. The workflow identity was assigned the **Azure
Kubernetes Service Cluster User Role** and **Azure Kubernetes Service
RBAC Writer** at the AKS cluster scope. The human operator was assigned
**Azure Kubernetes Service RBAC Reader** to view workloads in the Azure
portal.

Role assignments and federation can be configured through the Azure
portal or Azure CLI. Use least privilege and verify the scope before
assigning roles.

## Configure GitHub OIDC

1.  Create an Entra app registration/service principal for GitHub
    Actions.
2.  Add a federated credential for this repository's `main` branch. The
    subject should match the repository and branch exactly, for example
    `repo:OWNER/REPOSITORY:ref:refs/heads/main`.
3.  Add the app's client ID, tenant ID, and subscription ID as GitHub
    repository secrets.
4.  Assign the workflow identity the AKS Cluster User Role and Azure
    Kubernetes Service RBAC Writer at the required scope.
5.  Confirm that the workflow has `id-token: write` permission and uses
    `azure/login@v2`.

The workflow uses `kubelogin convert-kubeconfig -l azurecli` so
Kubernetes authentication uses the Azure login session.

## Run the lab

1.  Confirm the GitHub secrets and Azure OIDC configuration are correct.
2.  Ensure the resource group and AKS cluster exist and the workflow
    identity has the required permissions.
3.  Review `.github/workflows/ci.yml` and
    `.github/workflows/deploy-aks.yml`.
4.  Push an application change to `main`.
5.  Open **GitHub → Actions** and watch the CI workflow.
6.  CI validates the files, builds the images, scans them, publishes
    them to Docker Hub, and commits updated image tags to the Kubernetes
    manifest.
7.  The manifest commit triggers CD.
8.  Confirm the CD workflow succeeds and the frontend and Redis
    deployments roll out.
9.  Open the frontend service's external IP address in a browser.

### Image tagging

Images are tagged with the Git commit SHA, for example:

```text
DOCKERHUB_USERNAME/azure-vote-front:<commit-sha>
DOCKERHUB_USERNAME/azure-vote-redis:<commit-sha>
```

This makes the deployed image versions traceable to a source commit. The
CI workflow only publishes images on a push event; pull-request runs
validate/build/scan without publishing or committing a manifest update.

### Security behavior

The Trivy steps scan for `HIGH` and `CRITICAL` vulnerabilities and use
`exit-code: "1"`. With this configuration, a matching finding can fail
CI and prevent publishing and manifest updates. `ignore-unfixed: true`
means vulnerabilities without a fix are excluded from the failure
decision.

## Verify the deployment

From a shell authenticated to the cluster:

```bash
kubectl get nodes
kubectl get deployments
kubectl get pods -o wide
kubectl get services
kubectl rollout status deployment/azure-vote-front --timeout=180s
kubectl rollout status deployment/azure-vote-back --timeout=180s
kubectl get deployment azure-vote-front -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
kubectl get deployment azure-vote-back -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

The frontend Service is configured as `LoadBalancer`. Get its external
IP with:

```bash
kubectl get service azure-vote-front
kubectl get service azure-vote-front -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```

## Troubleshooting

See [`LAB-COMMANDS.md`](LAB-COMMANDS.md) for Azure CLI, Docker, Git,
kubectl, kubelogin, diagnostic, and cleanup commands.

Common checks:

- **OIDC login fails:** Verify the federated credential subject,
  issuer, audience, client ID, tenant ID, and GitHub secrets.
- **`kubectl` returns Unauthorized/Forbidden:** Confirm the kubeconfig
  was converted with `kubelogin`, and the workflow identity has the
  required AKS Azure RBAC role assignments.
- **Pods are `ImagePullBackOff`:** Check the exact image tag in the
  committed manifest, that the image exists in Docker Hub, and whether
  the repository is private.
- **Rollout fails:** Use `kubectl describe pod`, `kubectl logs`, and
  `kubectl get events`.
- **External IP stays pending:** Check the Service and AKS/Azure
  networking events; cloud load balancer provisioning may take several
  minutes.
- **CI fails at Trivy:** Review the scan output before deciding
  whether to fix, document, or change vulnerability policy. Do not
  simply disable scanning to make the pipeline green.
- **Manifest commit does not trigger CD:** Verify the commit changes
  `deployment/aks/azure-vote.yaml`, the CD workflow is enabled on the
  default branch, and workflow filters match the path and branch.

## Cost and cleanup

AKS nodes, load balancers, managed disks, public IP addresses, and other
resources may incur charges while running. Delete the lab resource group
when finished, after verifying it contains only resources you intend to
remove:

```bash
az group delete --name rg-azure-voting-lab --yes
az group exists --name rg-azure-voting-lab
```

The second command should return `false` once the resource group is
deleted. Review Cost Management for resources created outside the lab
resource group and for billing data that may appear later.

Deleting the Azure resource group does not delete the GitHub repository,
GitHub Actions workflows, GitHub secrets, or Docker Hub images.

## Future extension: GitOps with Argo CD

After the CI-to-manifest-commit-to-CD workflow is understood and stable,
the lab can be extended with Argo CD to continuously reconcile AKS with
the committed manifest. Argo CD is intentionally not required for this
version.

## References

- [Azure Kubernetes Service
  documentation](https://learn.microsoft.com/azure/aks/)
- [GitHub Actions documentation](https://docs.github.com/actions)
- [Azure Login GitHub Action and OIDC](https://github.com/Azure/login)
- [Trivy documentation](https://aquasecurity.github.io/trivy/)
- [Kubernetes documentation](https://kubernetes.io/docs/)

## License and attribution

This lab is based on the Azure Voting App pattern with a Python/Flask
frontend and Redis backend. Review the upstream repository's license and
retain applicable attribution when redistributing or modifying the
source.
