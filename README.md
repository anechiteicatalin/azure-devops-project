# azure-devops-project
1. Overview
This project uses GitHub Actions to implement a Continuous Deployment (CD) pipeline. Instead of using standard GitHub-hosted runners (which are ephemeral and lack access to private networks), this project utilizes a Self-Hosted Runner deployed on an Azure Virtual Machine.

Architecture Diagram
2. The Self-Hosted Runner (The Worker)
We configured a "Listening Bridge" where the Azure VM connects to GitHub to ask for jobs.

Host: Azure Virtual Machine (github-runner-vm) running Linux (Ubuntu).

Connection Method: Long Polling (Outbound HTTPS port 443).

Note: The VM initiates the connection out to GitHub. No inbound firewall ports were opened, maintaining high security.

Role: The runner executes the pipeline steps directly inside your Azure Virtual Network.

Dependencies Installed:

docker: To build the container image.

az cli: To authenticate with Azure.

kubectl: To send commands to the AKS Cluster.


3. The Pipeline Workflow (deploy.yml)
The file located at .github/workflows/deploy.yml defines the automation logic. Here is the technical breakdown of the steps:

Step,Command / Action,Description
Trigger,"on: push: branches: [ ""main"" ]",The pipeline starts automatically whenever code is pushed to the main branch.
Agent,runs-on: self-hosted,Critical: Tells GitHub to ignore its own cloud servers and send the job specifically to your Azure VM.
Checkout,actions/checkout@v3,Downloads your code from the repository onto the runner (the VM).
Login,azure/login@v1,Authenticates the runner using the Service Principal (Robot Account).
Build,docker build ...,Packages the python application into a Docker Image.
Push,docker push ...,Uploads the image to the Azure Container Registry (ACR) so the cluster can download it later.
Deploy,kubectl apply -f -,Connects to the AKS Cluster and commands it to pull the new image and restart the application pods.

4. Authentication & Secrets
We used two distinct methods for security, avoiding hardcoded passwords in the code.

A. The "Listener" Auth (Bridge 1)
Mechanism: Runner Token.

Setup: Generated via ./config.sh on the VM.

Purpose: Allows the VM to log in to GitHub and listen for "Queued" jobs.

B. The "Builder" Auth (Bridge 2)
Mechanism: Service Principal (SPN).

Storage: Saved in GitHub Secrets as AZURE_CREDENTIALS.

Format: JSON Object.

Purpose: Allows the pipeline (running on the VM) to authenticate with Azure to push images and update the cluster.

JSON

{
  "clientId": "...",
  "clientSecret": "...",
  "subscriptionId": "...",
  "tenantId": "..."
}
5. Known Issues & Troubleshooting
Documentation of errors encountered during setup and their resolutions:

"Permission denied (publickey)"

Cause: SSH Key mismatch between local machine and VM.

Fix: Reset VM credentials to a password-based login using az vm user update.

"Login failed: Content is not a valid JSON object"

Cause: Copy-pasting warning text along with the JSON output into GitHub Secrets.

Fix: Regenerated Service Principal and carefully copied only the { ... } block.

"command not found: az" or "kubectl"

Cause: The Self-Hosted Runner is a "blank slate" Linux server. It does not come with cloud tools pre-installed.

Fix: Manually SSH'd into the runner and installed azure-cli and kubectl.

"Could not create role assignment"

Cause: The Service Principal (Robot) only has Contributor access, not Owner access, so it cannot assign permissions to link ACR and AKS.

Fix: Ran the permission link command manually as the human administrator, then removed that step from the pipeline.
