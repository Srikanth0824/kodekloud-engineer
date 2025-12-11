## Task: VM Setup with Web Storage Integration
The Nautilus DevOps team is tasked with setting up an environment to host a static web application. The application will serve static content from an Azure Storage Account, and a Virtual Machine (VM) will be configured to fetch and display this content using Nginx. The resources must follow best practices for security, performance, and accessibility.

**Task Details:**

1. Create a Virtual Network (VNet) and Subnet:
    - Create a VNet named `xfusion-vnet` in the **East US** region.
    - Create a subnet named `xfusion-subnet` within the VNet for the VM.
2. Create an Azure Storage Account:
    - Create a storage account named `xfusionstor28218` in the **East US** region with **Locally-redundant storage (LRS)**.
    - Create a Blob container named `xfusion-container` in the storage account.
    - Upload the `index.html` file located at `/root` on the client host to the container `xfusion-container`.
    - **Ensure the Storage Account is private and not publicly accessible** by disabling public access for the storage account.
3) Create a Virtual Machine (VM):
    - Create a VM named `xfusion-vm` in the **East US** region.
    - Use the **datacenter-vnet** and subnet **datacenter-subnet** for the VM.
    - Authentication: Use `SSH public key` authentication. (Please select `use existing public key` option, create public-key locally and paste contents of `~/.ssh/id_rsa.pub`)
    - Install `Nginx` on the VM.
    - Configure `Nginx` to serve the `index.html` file from the Azure Storage Account `xfusionstor28218`.
4) Verify Setup:
    - Verify that the Nginx web server on the client host serves the `index.html` file correctly when accessing the VM's public IP address.
    - Ensure that the static content is fetched from the Storage Account `xfusionstor28218` and displayed as expected on the web page.

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
VNET_NAME="xfusion-vnet"
SUBNET_NAME="xfusion-subnet"
VM_NAME="xfusion-vm"
LOCATION="eastus"
STORAGE_ACCOUNT="xfusionstor28218"
CONTAINER="xfusion-container"
```

### **Step 3: Create VNet and Subnet**
```bash
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --location $LOCATION \
  --address-prefix 10.0.0.0/16 \
  --subnet-name $SUBNET_NAME \
  --subnet-prefix 10.0.1.0/24
```

### **Step 4: Create Private Storage Account and Blob Container**
Create storage acccount
```bash
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --allow-blob-public-access false
```
Create blob container
```bash
az storage container create \
  --account-name $STORAGE_ACCOUNT \
  --name $CONTAINER
```
Upload `index.html` file
```bash
az storage blob upload \
  --account-name $STORAGE_ACCOUNT \
  --container-name $CONTAINER \
  --name index.html \
  --file /root/index.html
```

### **Step 5: Create VM**
Generate SSH key on `azure-client` host
```bash
ssh-keygen
```
Create VM
```bash
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --public-ip-sku Standard \
  --storage-sku Standard_LRS \
  --location $LOCATION \
  --size Standard_B1s
```
Open port `80` for web access
```bash
az vm open-port \
  --port 80 \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME
```
Get VM public IP
```bash
VM_PUBLIC_IP=$(az vm show --resource-group $RESOURCE_GROUP --name $VM_NAME --show-details --query publicIps -o tsv)
```
SSH into the VM
```bash
ssh azureuser@$VM_PUBLIC_IP
```

### **Step 6: Install Nginx and Azure CLI**
```bash
sudo apt update
sudo apt install -y nginx
```
Install Azure CLI on VM (to pull blob content)
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```
Login (VM needs read access to storage blob)
```bash
az login
```
Set variables on VM
```bash
STORAGE_ACCOUNT="xfusionstor28218"
CONTAINER="xfusion-container"
```
Download blob file
```bash
az storage blob download \
  --account-name $STORAGE_ACCOUNT \
  --container-name $CONTAINER \
  --name index.html \
  --file ~/index.html
```
Move file to Nginx document root
```bash
sudo mv ~/index.html /var/www/html
```
Restart Nginx
```bash
sudo systemctl restart nginx
```

### **Step 7: Verify setup**
Using Web Browser:
- Navigate to `http://<VM-PUBLIC-IP>`
- You should see the webpage.