## Task: VM and ACR Integration for Storage
The Nautilus DevOps team needs to set up an Azure Virtual Machine (VM) to interact with an Azure Blob Storage container for storing and retrieving data. The team must create a private storage account, configure Blob Storage, and test the functionality.

Task:

1. Azure Virtual Machine Setup:
    - Create a VM named `nautilus-vm` in the `East US` region.
    - Authentication: Use `SSH public key` authentication. (Please select `use existing public key` option, create public-key locally and paste contents of `~/.ssh/id_rsa.pub`).
    - Install Docker and Azure CLI on the VM.
    - Pull the Docker image from the ACR and run it on the VM, ensuring it listens on port `80`.

2. Azure Container Registry (ACR) Setup:
    - Create an ACR named `nautilusacr25123` in the East US region.
    - The repository name should be `nautilus/python-app`.
    - Build the Docker image using the Dockerfile already given under `/root/pyapp`.
    - Push the Docker image to the ACR with the tag `latest`.

3. Create a Storage Account and Blob Container:
    - Create a storage account named `nautilusstor25123` in the East US region with `Locally-redundant storage (LRS)`.
    - Create a Blob container named `nautilus-config`.
    - Upload a file named `config.json` (available under `/root/`) to the container.

4. Validation:
    - Confirm that the application can be accessed on the browser.

---

## Solution

We'll be performing this task using Azure CLI.

### **Step 1: Login to Azure CLI**
**From `azure-client` host**
```bash
az login
```
Follow the instructions and ensure that you are logged in.

### **Step 2: Set Variables**
Define variables for easier management:
```bash
RESOURCE_GROUP=$(az group list --query "[0].name" -o tsv)
VM_NAME="nautilus-vm"
LOCATION="eastus"
ACR_NAME="nautilusacr25123"
ACR_REPO="nautilus/python-app"
STORAGE_ACCOUNT="nautilusstor25123"
CONTAINER="nautilus-config"
LOCAL_CONFIG_FILE="/root/config.json"
DOCKERFILE_PATH="/root/pyapp"
```

### **Step 3: Azure Container Registry (ACR) Setup**
Create ACR
```bash
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Basic \
  --location $LOCATION
```

Login to ACR
```bash
az acr login --name $ACR_NAME
```

### **Step 4: Build and Push Docker Image**
```bash
# Build Docker image
docker build -t $ACR_NAME.azurecr.io/$ACR_REPO:latest $DOCKERFILE_PATH

# Push Docker image
docker push $ACR_NAME.azurecr.io/$ACR_REPO:latest
```

### **Step 5: Create Storage Account & Blob Container**
Create Storage Account
```bash
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

Get connection string
```bash
STORAGE_CONN=$(az storage account show-connection-string \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query connectionString -o tsv)
```

Create blob container
```bash
az storage container create \
  --name $CONTAINER \
  --connection-string "$STORAGE_CONN"
```

Upload `config.json` to container
```bash
az storage blob upload \
  --container-name $CONTAINER \
  --name config.json \
  --file $LOCAL_CONFIG_FILE \
  --connection-string "$STORAGE_CONN"
```

### **Step 6: Create Azure VM with SSH Key Authentication**
Generate SSH key on `azure-client` host
```bash
ssh-keygen

# Copy the public key:
PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
```

Create VM using existing public key
```bash
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS \
  --location $LOCATION \
  --admin-username azureuser \
  --ssh-key-value "$PUB_KEY"
```

Open port 80 for web access
```bash
az vm open-port --resource-group $RESOURCE_GROUP --name $VM_NAME --port 80
```

### **Step 7: Install Docker & Azure CLI on VM**
SSH into VM
```bash
VM_IP=$(az vm show --resource-group $RESOURCE_GROUP --name $VM_NAME --show-details --query publicIps -o tsv)
ssh azureuser@$VM_IP
```

**From `nautilus-vm` host**  
Install Docker and Azure CLI
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Logout and back in to refresh Docker group permissions
exit
```

### **Step 8: Run Docker container exposing port 80 on VM**
SSH to the VM
```bash
ssh azureuser@$VM_IP
```

Login to use Azure CLI
```bash
az login
```

Set varilables
```bash
STORAGE_ACCOUNT="nautilusstor25123"
CONTAINER="nautilus-config"
ACR_NAME="nautilusacr25123"
ACR_REPO="nautilus/python-app"
```

Download `config.json` file from the Blob Container
```bash
az storage blob download \
  --account-name $STORAGE_ACCOUNT \
  --container-name nautilus-config \
  --name config.json \
  --file /home/azureuser/config.json
```  

Login to ACR to pull the image
```bash
az acr login --name $ACR_NAME
```

Run docker container
```bash
docker run -d --name app \
  -p 80:80 \
  -v /home/azureuser/config.json:/app/config.json \
  $ACR_NAME.azurecr.io/$ACR_REPO:latest
```

### **Step 9: Verify setup**
Using `curl` from the `nautilus-vm` itself
```bash
curl localhost:80
```

Using Web Browser:
- Navigate to `http://<VM-PUBLIC-IP>`
- You should see the webpage.