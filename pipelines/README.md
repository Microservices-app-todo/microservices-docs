# 🚀 Release Pipeline (General)
This pipeline is a **GitHub Action** workflow designed to automate the release process using **Semantic Release**. It is triggered on every push to the `develop` branch and follows a predefined configuration to analyze commits, generate release notes, and publish releases to GitHub.

## 📋 Requirements

Before using this pipeline, make sure your project meets the following requirements:

- **Node.js version**: Must be **greater than 20.8.1**
- **Semantic Release configuration file** (usually named `.releaserc.json`, `release.config.js`, or in `package.json`)
- **GitHub Token**: Must be available in your repository’s secrets as `GITHUB_TOKEN` (automatically provided by GitHub Actions)

## ⚖️ Configuration

This pipeline uses the following Semantic Release configuration:

```json
{
  "branches": ["develop"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/github"
  ]
}
```

### Explanation of Plugins:

- **@semantic-release/commit-analyzer**: Determines the type of version bump based on conventional commits (e.g., `feat`, `fix`, `BREAKING CHANGE`)
- **@semantic-release/release-notes-generator**: Generates release notes from the commits
- **@semantic-release/github**: Publishes the release directly to the GitHub repository

## ⚙️ Workflow Breakdown

```yaml
name: Release

on:
  push:
    branches:
      - develop

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22  # Ensure this is > 20.8.1

      - name: Install dependencies
        run: npm ci

      - name: Run Semantic Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx semantic-release
```

### ✅ What it does:

1. **Checks out** the source code
2. **Installs Node.js** (version 22, which satisfies the version requirement)
3. **Installs dependencies** using `npm ci`
4. **Runs Semantic Release**, which:
   - Analyzes commits
   - Determines the next version
   - Generates release notes
   - Publishes the release to GitHub


# Infrastructure Pipeline Setup Guide

This guide outlines the necessary steps and secrets required to configure an infrastructure pipeline. It includes credentials and configurations needed for services like GitHub, DockerHub, and Azure.

---

## 🔐 Required Secrets and Tokens

### 1. **GitHub Personal Access Token (PAT)**
- **Use case**: Allows CI/CD pipelines (e.g., GitHub Actions) to access private repositories.
- **How to create**:
  1. Go to GitHub > Settings > Developer settings > Personal access tokens.
  2. Generate a new token with appropriate scopes (e.g., `repo`, `workflow`).
  3. Save it securely.
- **Where to use**:
  - Store it as a secret in GitHub Actions (Settings > Secrets and variables > Actions).
  - Name it something like `GH_PERSONAL_ACCESS_TOKEN`.

---

### 2. **DockerHub Access Token**
- **Use case**: Prevents hitting DockerHub's pull image quota in CI/CD.
- **How to create**:
  1. Go to DockerHub > Account Settings > Security.
  2. Create a new access token.
- **Where to use**:
  - Save it as `DOCKERHUB_TOKEN` in your secrets store (e.g., GitHub Actions).
  - Also save the DockerHub username as `DOCKERHUB_USERNAME`.

---

### 3. **Azure Service Principal Token**
- **Use case**: Provides Azure CLI and scripts with permission to deploy infrastructure.
- **Step 1: Create the Service Principal**
  ```bash
  az ad sp create-for-rbac --name "infra-sp" --role="Owner" \
    --scopes "/subscriptions/7c45ba24-1d27-4c6a-a5bd-1d7b9a655200/resourceGroups/microservices-rg"
  ```
- **Step 2: Assign Role (if needed separately)**
  ```bash
  az role assignment create \
    --assignee b445af1d-68d3-4bc2-9a38-ce4f9659d8fe \
    --role "Owner" \
    --scope /subscriptions/7c45ba24-1d27-4c6a-a5bd-1d7b9a655200/resourceGroups/microservices-rg
  ```
- **What you get** (Example):
  ```json
  {
    "clientId": "541ea040-xxxx-xxxx",
    "clientSecret": "xxxxx",
    "subscriptionId": "7c45ba24-xxxx",
    "tenantId": "61xxb"
  }
  ```
- **Where to use**:
  - Store in your secrets as:
    - `AZURE_CLIENT_ID`
    - `AZURE_CLIENT_SECRET`
    - `AZURE_SUBSCRIPTION_ID`
    - `AZURE_TENANT_ID`
  - These are used in Terraform, Pulumi, Azure CLI, etc.

---

## 📦 Secret Configuration Example (GitHub Actions)

```yaml
env:
  GH_TOKEN: ${{ secrets.GH_PERSONAL_ACCESS_TOKEN }}
  DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
  DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
  AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
  AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
```

---

## ⚙️ Infrastructure Pipeline Flow

This section describes a typical CI/CD flow for deploying infrastructure using the previously configured secrets and tools:

### 1. **Trigger**
- Push to `main` branch or PR merge triggers the GitHub Actions workflow.

### 2. **Checkout Code**
- GitHub Actions uses `actions/checkout@v3` to pull the latest code.

### 3. **Authenticate to Azure**
- Azure CLI is configured with:
  ```bash
  az login --service-principal \
    -u $AZURE_CLIENT_ID \
    -p $AZURE_CLIENT_SECRET \
    --tenant $AZURE_TENANT_ID
  ```

### 4. **Pull Docker Images**
- Log in to DockerHub to avoid rate limits:
  ```bash
  echo $DOCKERHUB_TOKEN | docker login -u $DOCKERHUB_USERNAME --password-stdin
  ```

### 5. **Run Terraform/Pulumi/ARM Deployment**
- Run infrastructure as code using your preferred tool:
  ```bash
  terraform init
  terraform apply -auto-approve
  ```
  or
  ```bash
  pulumi up --yes
  ```

### 6. **Optional: Push Docker Images to Azure Container Registry (ACR)**
- Tag and push image:
  ```bash
  docker tag myimage:latest myacr.azurecr.io/myimage:latest
  docker push myacr.azurecr.io/myimage:latest
  ```

### 7. **Post-deployment Validation**
- Ping services, run smoke tests, and notify via Slack/Discord.

---

## ✅ Summary Checklist
- [ ] GitHub PAT created and stored.
- [ ] DockerHub token and username stored.
- [ ] Azure service principal created and stored.
- [ ] All secrets added to GitHub Actions or your CI/CD secrets manager.
- [ ] Infrastructure-as-code (IaC) tools configured to use these secrets.
- [ ] Pipeline steps implemented and tested.

---

You’re now set to securely automate infrastructure deployment with full control over permissions and image pulls.

## GitHub Actions Pipeline: Build and Push Docker Image to Azure Container Registry (ACR)

This GitHub Actions pipeline automates the process of building a Docker image from your repository, pushing it to an Azure Container Registry (ACR), and deploying it to an Azure Container App. It is triggered manually (`workflow_dispatch`) or when code is pushed to the `develop` branch.

### ❗ Dependencies
This pipeline assumes that an **infrastructure pipeline** has already set up the necessary **secrets** in your GitHub repository. These secrets include:
- `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`: For Docker Hub login.
- `DOCKER_USERNAME`, `DOCKER_PASSWORD`: For Azure Container Registry login.
- `ACR_NAME`: The name of your ACR instance (e.g., `myregistry` if your login server is `myregistry.azurecr.io`).
- `CREDS`: Azure service principal credentials in JSON format.
- `CONTAINER_APP_NAME`: Name of your Azure Container App.
- `RESOURCE_GROUP`: Name of the Azure Resource Group where your Container App is deployed.

---

### 🚀 Job: `build-and-deploy`
Runs on `ubuntu-latest` and includes the following steps:

#### 1. **Checkout code**
Pulls the code from the repository.
```yaml
- name: Checkout code
  uses: actions/checkout@v2
```

#### 2. **Login to Docker Hub**
Authenticates to Docker Hub using provided credentials.
```yaml
- name: Log in to Docker Hub
  uses: docker/login-action@v2
```

#### 3. **Setup Docker Buildx**
Prepares Docker Buildx for building multi-platform images.
```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v2
```

#### 4. **Login to Azure Container Registry (ACR)**
Authenticates to your private ACR using secrets.
```yaml
- name: Log in to Azure Container Registry
  uses: azure/docker-login@v1
```

#### 5. **Build and push Docker image**
Builds the Docker image from the current directory and pushes it to the specified ACR.
```yaml
- name: Build and push local Docker image
  uses: docker/build-push-action@v2
```

#### 6. **Login to Azure (conditionally)**
Logs in to Azure using service principal credentials only when the workflow is triggered by a push.
```yaml
- name: Log in to Azure
  if: github.event_name == 'push'
  uses: azure/login@v1
```

#### 7. **Update Azure Container App (conditionally)**
Triggers a new revision of the Azure Container App with the latest image.
```yaml
- name: Force new revision of Azure Container App
  if: github.event_name == 'push'
  run: az containerapp update ...
```

---

### 🧩 Summary
This pipeline ensures that any updates to the `develop` branch are:
- Built into a Docker image
- Pushed to ACR
- Deployed to the Azure Container App (on push only)

It is tightly coupled with the infrastructure pipeline that must handle secret provisioning beforehand.