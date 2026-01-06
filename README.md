# 🚀 Azure DevOps Project: End-to-End CI/CD

![Azure](https://img.shields.io/badge/azure-%230072C6.svg?style=for-the-badge&logo=microsoftazure&logoColor=white) ![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white) ![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white) ![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)

## 📖 Overview
This project implements a complete **Continuous Deployment (CD) pipeline** using GitHub Actions.

Instead of relying on standard GitHub-hosted runners (which are ephemeral and lack access to private networks), this project utilizes a **Self-Hosted Runner** deployed on a private Azure Virtual Machine. This architecture mimics a real-world enterprise environment where security and network control are paramount.

---

## 🏗️ Architecture

### 1. The Diagram
![Architecture Diagram](https://github.com/anechiteicatalin/azure-devops-project/blob/main/image.png?raw=true)
*(Note: Ensure you upload your diagram image to an `images` folder in your repo)*

### 2. The Self-Hosted Runner (The Worker) 🤖
We configured a **"Listening Bridge"** where the Azure VM connects to GitHub to ask for jobs.

* **Host:** Azure Virtual Machine (`github-runner-vm`) running Linux (Ubuntu).
* **Connection Method:** Long Polling via outbound HTTPS (Port 443).
* **Role:** The runner executes pipeline steps directly inside the Azure Virtual Network.

> **🔐 Security Note:** The VM initiates the connection *out* to GitHub. No inbound firewall ports were opened to the public internet, maintaining high security.

**Dependencies Installed on Runner:**
* 🐳 **docker:** To build the container image.
* ☁️ **az cli:** To authenticate with Azure resources.
* ☸️ **kubectl:** To send deployment commands to the AKS Cluster.

---

## ⚙️ The Pipeline Workflow
The automation logic is defined in `.github/workflows/deploy.yml`.

| Step | Command / Action | Description |
| :--- | :--- | :--- |
| **Trigger** | `on: push: branches: [ "main" ]` | The pipeline starts automatically whenever code is pushed to the `main` branch. |
| **Agent** | `runs-on: self-hosted` | **Critical:** Tells GitHub to ignore its own cloud servers and send the job specifically to our private Azure VM. |
| **Checkout** | `actions/checkout@v3` | Downloads the source code from the repository onto the runner (the VM). |
| **Login** | `azure/login@v1` | Authenticates the runner using the Service Principal (Robot Account). |
| **Build** | `docker build ...` | Packages the Python application into a Docker Image. |
| **Push** | `docker push ...` | Uploads the image to the **Azure Container Registry (ACR)**. |
| **Deploy** | `kubectl apply -f -` | Connects to the **AKS Cluster**, pulls the new image, and updates the application pods. |

---

## 🔐 Authentication & Secrets
We utilize two distinct security bridges to avoid hardcoding passwords.

### Bridge 1: The "Listener" Auth
* **Mechanism:** Runner Token
* **Setup:** Generated via `./config.sh` on the VM during setup.
* **Purpose:** Allows the VM to log in to GitHub and "listen" for queued jobs.

### Bridge 2: The "Builder" Auth
* **Mechanism:** Service Principal (SPN)
* **Storage:** Saved in GitHub Secrets as `AZURE_CREDENTIALS`.
* **Purpose:** Allows the pipeline script (running on the VM) to authenticate with Azure to manage resources.

**JSON Structure:**
```json
{
  "clientId": "<GUID>",
  "clientSecret": "<GUID>",
  "subscriptionId": "<GUID>",
  "tenantId": "<GUID>"
}
