## Task: Create a Public IP Address for Azure VM
The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the Azure cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition. To achieve this, they have segmented large tasks into smaller, more manageable units. This granular approach enables the team to execute the migration in gradual phases, ensuring smoother implementation and minimizing disruption to ongoing operations. By breaking down the migration into smaller tasks, the Nautilus DevOps team can systematically progress through each stage, allowing for better control, risk mitigation, and optimization of resources throughout the migration process.

For this task, allocate a `Public IP` address, name it as `xfusion-pip`.

---

## Solution

### **Step 1: Log in to Azure Portal**
Go to the Azure Portal:  
https://portal.azure.com  
Sign in with the credentials provided.

### **Step 2: Search for Public IP Addresses**
- In the top search bar, type **Public IP addresses**.  
- Select **Public IP addresses** from the list.  
![search public ip](assets/day7_01.png)

### **Step 3: Create a New Public IP Address**
- Click **Create** 

### **Step 4: Fill in the Basics Section**
Provide value(s) in the **Basics** tab as per the task description:
  
- **Name:** `xfusion-pip`  
- **IP Version:** `IPv4` (default)  
![public ip basics](assets/day7_02.png)

Leave other fields as default.

### **Step 5: Review and Create**
- Click **Review + create**  
- Wait for validation to complete  
- Review all the configuration settings  
- Click **Create** to allocate the Public IP address  
![review and create](assets/day7_03.png)

Azure will now provision the Public IP address.
