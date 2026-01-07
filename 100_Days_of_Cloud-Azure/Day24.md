## Task: Securing Virtual Machine SSH Access
The Nautilus DevOps team needs to set up a new Virtual Machine (VM) on the Azure cloud that can be accessed securely from their landing host (`azure-client`). 

1. **Create an SSH Key:** On the `azure-client` host, check if an SSH key already exists. If it doesn't exist, create a new SSH key on the `azure-client` host that will be used for password-less SSH access.
2. **Create a Virtual Machine:** Use the Azure Portal or Azure CLI to create a new Virtual Machine named `xfusion-vm` in the `westus` region. Set the VM size to Standard_B1s and configure the VM with SSH access for the `azureuser` account using the newly created SSH key.
3. **Configure SSH Access:** Ensure that the SSH key from the `azure-client` host is added to the `azureuser` account on `xfusion-vm`, enabling secure, password-less SSH access from the `azure-client` host.
4. **Verify Connectivity:** Test the connection from `azure-client` to `xfusion-vm` using SSH to confirm that password-less access has been set up correctly.

---

## Solution

### **Step 1: Check and Create SSH Key on Azure Client Host**

```bash
# Check if SSH keys exist in the default location
ls -la ~/.ssh/
```

**Look for files named:**
- `id_rsa` (private key)
- `id_rsa.pub` (public key)

If the key exists and you're happy to use it. Otherwise, create a new key.

```bash
# Generate a new SSH key pair with default values
ssh-keygen
```

### **Step 2: Copy SSH public key**
Copy the public SSH key, we'll use it while creating the VM in next step.
```bash
cat ~/.ssh/id_rsa.pub
```

### Step 3: Create VM from portal
Refer to [Day02](Day02.md) to create a VM from Azure portal, only difference is in the ssh key section.  
**In Administrator Account Section**
- **Authentication type:** `SSH public key`  
- **Username:** `azureuser`  
- **SSH public key source:** `Use existing public key`  
- **SSH public key:** Paste the content from `~/.ssh/id_rsa.pub`  
![ssh key](assets/day24_01.png)

### **Step 4: Review and Create VM**
- Review all configuration settings:
  - VM name: **xfusion-vm**
  - Size: **Standard_B1s**
  - Public IP: **xfusion-pip (Static)**
  - SSH authentication configured
- Click **Review + create**  
- Wait for validation  
- Click **Create**  

Azure will now provision the Virtual Machine. This may take 3-5 minutes.

### **Step 5: SSH into the VM**
Once deployment completes:

- Go to **Virtual Machines → xfusion-vm**  
- Copy the public IP address  
- SSH into the VM from a terminal:

```bash
ssh azureuser@<PublicIP>
```
## CLI Solution
```bash
az vm create \
  -g kml_rg_main-19157c42f5c34283 \ # replace it with your resource group name
  -n datacenter-vm \
  -l westus \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub 
```
