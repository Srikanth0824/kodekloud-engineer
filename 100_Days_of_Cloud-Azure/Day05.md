## Task: Create a Virtual Network (IPv4) in Azure
The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the Azure cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition. To achieve this, they have segmented large tasks into smaller, more manageable units. This granular approach enables the team to execute the migration in gradual phases, ensuring smoother implementation and minimizing disruption to ongoing operations.

Create a Virtual Network (VNet) named `datacenter-vnet` in the `East US` region with `192.168.0.0/24` IPv4 CIDR.

---

## Solution

### **Step 1: Log in to Azure Portal**
Go to the Azure Portal:  
https://portal.azure.com  
Sign in with the credentials provided.

### **Step 2: Search for Virtual Networks**
- In the top search bar, type **Virtual Networks**.  
- Select **Virtual Networks** from the list.  
![search vnet](assets/day4_01.png)

### **Step 3: Create a New Virtual Network**
- Click **Create** 

### **Step 4: Fill in the Basics Section**
Provide the following values:
  
- **Resource Group:** Select an existing resource group or create a new one  
- **Virtual network name:** `datacenter-vnet`  
- **Region:** `East US`  
![vnet basics](assets/day5_02.png)

### **Step 5: Configure IP Address Space**
In the **IP Addresses** tab:

- **IPv4 address space:** Specify `192.168.0.0/24` IPv4 CIDR block  
- **Subnet:** A default subnet will be created automatically (e.g., `default` with `192.168.0.0/24`)  
![ip address space](assets/day5_03.png)

### **Step 6: Review and Create**
- Review all the configuration settings  
- Click **Review + create**  
- Wait for validation to complete  
- Click **Create** to deploy the Virtual Network  

Azure will now provision the Virtual Network.