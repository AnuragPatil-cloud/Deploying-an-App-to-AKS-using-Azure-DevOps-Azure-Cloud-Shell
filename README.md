# k8s-application

A small Node.js (Express + Pug) demo app, containerized with Docker and shipped to Kubernetes — with a full walkthrough for deploying it to **Azure Kubernetes Service (AKS)** using **Azure DevOps** and **Azure Cloud Shell**.

This repo pairs the runnable demo app with the end-to-end deployment guide so both live in one place: clone it, build it, and deploy it your way — a quick local Docker run, a plain `kubectl apply` to any cluster, or the full CI/CD pipeline into AKS.

## Tech Stack

- **App**: Node.js, Express, Pug
- **Container**: Docker
- **Orchestration**: Kubernetes (Deployment + LoadBalancer Service)
- **Registry**: Docker Hub ([anuragpatilcloud](https://hub.docker.com/u/anuragpatilcloud)) or Azure Container Registry
- **CI/CD**: Azure DevOps Pipelines
- **Cluster**: Azure Kubernetes Service (AKS)

## Project Structure

```
k8s-application/
├── app.js                  # Express app entrypoint
├── package.json
├── Dockerfile
├── views/
│   └── home.pug            # Landing page template
├── manifests/
│   ├── deployment.yml      # Kubernetes Deployment
│   └── service.yml         # Kubernetes LoadBalancer Service
├── azure-pipelines.yml     # Azure DevOps pipeline (build → push → deploy)
└── README.md
```

## Quick Start — Run Locally with Docker

```bash
git clone https://github.com/AnuragPatil-cloud/k8s-application.git
cd k8s-application

docker build -t anuragpatilcloud/k8s-application:latest .
docker run -p 8888:8888 anuragpatilcloud/k8s-application:latest
```

Visit **http://localhost:8888**.

## Push the Image to Docker Hub

```bash
docker login
docker push anuragpatilcloud/k8s-application:latest
```

## Deploy to Any Kubernetes Cluster

The manifests in `manifests/` already point at the Docker Hub image above, so once it's pushed you can deploy to any cluster you have `kubectl` access to:

```bash
kubectl apply -f manifests/deployment.yml
kubectl apply -f manifests/service.yml

# Get the external IP once the LoadBalancer is provisioned
kubectl get service k8s-application
```

Open `http://<EXTERNAL-IP>:8888` in your browser.

---

## Full Walkthrough: CI/CD to AKS via Azure DevOps & Azure Cloud Shell

The section below walks through standing up the infrastructure and CI/CD pipeline so every push to `master` automatically builds, pushes, and redeploys the app on AKS.

### Prerequisites

1. Access to an Azure account
2. Access to Azure DevOps and a PAT token
3. Access to a GitHub account
4. An Azure DevOps organization — create one [here](https://aex.dev.azure.com/) via "Create a new organization"
5. All commands below are run in [Azure Cloud Shell](https://shell.azure.com/) — type `bash` in the terminal to switch to Bash if it defaults to PowerShell

### Overall Architecture

[![Overall Architecture](https://res.cloudinary.com/practicaldev/image/fetch/s--aST7vxoo--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/9yjcvrdm1uibekjf354m.PNG)](https://res.cloudinary.com/practicaldev/image/fetch/s--aST7vxoo--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/9yjcvrdm1uibekjf354m.PNG)

*Diagram created with Cloud Skew (free) — worth checking out.*

- **Azure DevOps & GitHub** handle source control and CI/CD: your application, infrastructure, and pipeline code live in this GitHub repo, and the CI/CD pipeline is an Azure YAML pipeline.
- **Azure Container Registry (ACR)** is Azure's native container registry (comparable to Docker Hub) — the pipeline can build and push the image here, with a new version created on every successful run.
- **Azure Kubernetes Service (AKS)** is Azure's managed Kubernetes offering — it's what actually runs the Deployment and Service defined in `manifests/`.
- **Azure Active Directory** provides the Service Principal used to create a secure, identity-based Service Connection so the pipeline can deploy to the right subscription with the right permissions.

### Initial Setup

Add the Azure DevOps extension to your Cloud Shell session:

```bash
az extension add --name azure-devops
```

Point your shell at your DevOps organization:

```bash
az devops configure --defaults organization=https://dev.azure.com/insertorgnamehere/
```

Set the `AZURE_DEVOPS_EXT_PAT` environment variable so you don't have to sign in explicitly for every command:

```bash
export AZURE_DEVOPS_EXT_PAT=insertyourpattokenhere
```

Create a new Azure DevOps project:

```bash
az devops project create --name k8s-project
```

Set it as the default project:

```bash
az devops configure --defaults project=k8s-project
```

### Deploying the Infrastructure

Create a resource group:

```bash
az group create --location westeurope --resource-group my-aks-rg
```

Create a service principal — **copy the output, you'll need it next**:

```bash
az ad sp create-for-rbac --skip-assignment
```

Create the AKS cluster using the service principal output above:

> If you hit `400 Client Error: Bad Request for url` on the role assignment, it's a [known Azure CLI issue](https://github.com/Azure/azure-cli/issues/9345) — just re-run the command.

```bash
az aks create -g my-aks-rg -n myakscluster -c 1 --generate-ssh-keys --service-principal "insertappidhere" --client-secret "insertpasswordhere"
```

Create an Azure Container Registry:

```bash
az acr create -g my-aks-rg -n insertuniqueacrnamehere --sku Basic --admin-enabled true
```

Grant AKS pull access to the ACR:

```bash
ACR_ID=$(az acr show --name insertuniqueacrnamehere --resource-group my-aks-rg --query "id" --output tsv)

CLIENT_ID=$(az aks show -g my-aks-rg -n myakscluster --query "servicePrincipalProfile.clientId" --output tsv)

az role assignment create --assignee $CLIENT_ID --role acrpull --scope $ACR_ID
```

### Deploying the Application

Since the app code already lives in **this** repo (`AnuragPatil-cloud/k8s-application`), there's no separate fork step — just clone your own repo into the Cloud Shell session:

```bash
git clone https://github.com/AnuragPatil-cloud/k8s-application.git
cd k8s-application
```

Create a pipeline in Azure DevOps:

```bash
az pipelines create --name "k8s-application-pipeline"
```

Follow the prompts:

1. Enter your GitHub username; press enter
2. Enter your GitHub password; press enter
3. Confirm your GitHub password again; press enter
4. Enter your two-factor code, if enabled
5. Enter a service connection name (e.g. `k8sapplicationpipeline`); press enter
6. Choose **[3]** to deploy to Azure Kubernetes Service
7. Select the AKS cluster you just created
8. Choose **[2]** for the `default` Kubernetes namespace
9. Select the ACR you just created
10. Accept the default image name (press enter)
11. Accept the default service port (press enter)
12. Leave "enable review app flow for pull requests" blank; press enter
13. Choose **[1]** to continue with the generated YAML
14. Choose **[1]** to commit directly to the `master` branch

This regenerates `azure-pipelines.yml` in your repo with your real service connection, ACR, and environment names.

### Congratulations 🎉

Give it a few minutes to build the container, push it to ACR, and deploy to AKS.

Get your `kubectl` credentials:

```bash
az aks get-credentials --resource-group my-aks-rg --name myakscluster
```

Check the resources the pipeline created:

```bash
kubectl get all
```

[![kubectl get all](https://res.cloudinary.com/practicaldev/image/fetch/s--kSIvGnYa--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/5ckrh8renuq0nerc3j88.PNG)](https://res.cloudinary.com/practicaldev/image/fetch/s--kSIvGnYa--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/5ckrh8renuq0nerc3j88.PNG)

Copy the External IP from the service and open `http://<EXTERNAL-IP>:8888` — you should see the demo app running live from AKS:

[![Super k8s Demo](https://res.cloudinary.com/practicaldev/image/fetch/s--IlcXQ_4m--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/rak18btkdex17vlu4my7.PNG)](https://res.cloudinary.com/practicaldev/image/fetch/s--IlcXQ_4m--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/rak18btkdex17vlu4my7.PNG)

### Summary

At this point you've stood up a new Azure DevOps project with a working CI/CD pipeline: it builds the app into a container, pushes it to a registry, and deploys it to AKS — all triggered automatically on every push to `master` (see the `trigger: - master` line in `azure-pipelines.yml`).

## Credits

- Base demo application (`app.js`, `views/`, original Dockerfile) originally created by [Andrew Urwin](https://github.com/ghostinthewires) — [ghostinthewires/k8s-application](https://github.com/ghostinthewires/k8s-application)
- Docker Hub packaging, Kubernetes manifests, and the AKS / Azure DevOps deployment guide by Anurag Patil

## 🛠️ Author & Community

This project is maintained by **[Anurag Patil](https://github.com/AnuragPatil-cloud)** 💡.
Feedback and contributions are welcome — feel free to open an issue or PR.

📧 **Connect with me:**

- **GitHub**: https://github.com/AnuragPatil-cloud
- **Docker Hub**: https://hub.docker.com/u/anuragpatilcloud
