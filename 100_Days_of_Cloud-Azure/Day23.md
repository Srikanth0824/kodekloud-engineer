## Task: Automating User Data Configuration Using the CLI
The Nautilus DevOps Team is working on setting up a new virtual machine (VM) to host a web server for a critical application. The team lead has requested you to create an Azure VM that will serve as a web server using Nginx. This VM will be part of the initial infrastructure setup for the Nautilus project. Ensuring that the server is correctly configured and accessible from the internet is crucial for the upcoming deployment phase.

As a member of the Nautilus DevOps Team, your task is to create a VM using Azure CLI with the following specifications:

1. **Instance Name:** The VM must be named `xfusion-vm`.
2. **Image:** Use any available Ubuntu image to create this VM.
3. **Custom Script Extension/User Data:** Configure the VM to run a custom script during its launch. This script should:
   - Install the Nginx package.
   - Start the Nginx service.
4. **Network Security Group (NSG):** Ensure that the VM allows HTTP traffic on port `80` from the internet.

**Instructions:**
- Use **Azure CLI** commands to set up the VM in the specified configuration.
- Ensure the VM is accessible from the internet on port **80**.
- The **Nginx** service should be running after setup.

---

## Solution

### **Step 1: Prepare user data file**
Before creating the VM, prepare the user-data script that will install and configure Nginx.
```bash
cat > userdata.txt << 'EOF'
#!/bin/bash
sudo apt-get update
sudo apt-get install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
EOF
```

### **Step 2: Get the existing resource group name**
```bash
az group list --query "[0].name" -o tsv
```
Use this output as value for **RESOURCE_GROUP** in the next step.  

### **Step 3: Set Variables**
Define variables for easier management:
```bash
# Set variables
RESOURCE_GROUP="kml_rg_main-642178379067476f"
LOCATION="eastus"
VM_NAME="nautilus-vm"
IMAGE="Ubuntu2204"
SIZE="Standard_B1s"
ADMIN_USERNAME="azureuser"
NSG_NAME="${VM_NAME}-nsg"
```

### **Step 4: Create the Virtual Machine with UserData**
```bash
# Create VM with user data
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --location $LOCATION \
  --image $IMAGE \
  --size $SIZE \
  --admin-username $ADMIN_USERNAME \
  --generate-ssh-keys \
  --user-data userdata.txt \
  --public-ip-sku Standard \
  --nsg $NSG_NAME \
  --nsg-rule SSH \
  --storage-sku Standard_LRS
```

**Command Explanation:**
- `--resource-group`: Resource group name
- `--name`: VM name (xfusion-vm)
- `--image`: Ubuntu 22.04 LTS image
- `--size`: VM size (Standard_B1s)
- `--admin-username`: Administrator username
- `--generate-ssh-keys`: Automatically generate SSH keys
- `--user-data`: UserData configuration file
- `--public-ip-sku Standard`: Use Standard SKU for public IP
- `--nsg`: Network Security Group name
- `--nsg-rule SSH`: Create NSG rule for SSH (port 22)
- `--storage-sku`: Sets OS disk SKU

**Note the public IP address** - you'll need it to access the web server to verify if Nginx service is running.
This process will take 3-5 minutes to complete.

### **Step 5: Open Port 80 for HTTP Traffic**
Add an inbound NSG rule to allow HTTP traffic on port 80:
```bash
# Open port 80 for HTTP traffic
az vm open-port \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --port 80 \
  --priority 1001
```

### **Step 6: Get the Public IP Address**
Retrieve the public IP address of the VM:
```bash
# Get public IP address
PUBLIC_IP=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query publicIps \
  --output tsv)

echo "VM Public IP Address: $PUBLIC_IP"
echo "Access the web server at: http://$PUBLIC_IP"
```

### **Step 7: Test Web Server from Internet**
Test the web server accessibility from your `azure-client` host's terminal:
```bash
# Test using curl
curl http://$PUBLIC_IP
```
