## Task
The Nautilus DevOps team is migrating services to Azure. They are breaking down tasks to ensure better control and optimization. You are tasked with attaching an existing network interface (NIC) to a virtual machine (VM).

An existing VM named `datacenter-vm` and a network interface named `datacenter-nic` already exist in the `West US` region.

- Attach the network interface `datacenter-nic` to the VM `datacenter-vm`.
- Ensure the NIC's status is `attached` before submitting the task.

Make sure that the virtual machine initialization has been completed before submitting this task.

---

## Solution

#### **Step 1: Log in to Azure Portal**
Go to the Azure Portal:  
https://portal.azure.com  
Sign in with the credentials provided.

#### **Step 2: Stop the Virtual Machine**
**Important:** Before attaching a network interface to a VM, the VM must be in a **stopped (deallocated)** state.

- In the top search bar, type **Virtual Machines**.  
- Select **Virtual Machines** from the list.  
- Click on **datacenter-vm**.  
- Click **Stop** at the top of the page.  
- Wait for the VM status to show **Stopped (deallocated)**.  
![stop vm](assets/day9_01.png)

#### **Step 3: Navigate to Networking Section**
- Once the VM is stopped, in the left-hand menu under **Networking**, click on **Network settings**.  
- In the **Network settings** page, click on **Attach network interface**.  
![navigate to networking](assets/day9_02.png)

#### **Step 4: Select and attach the Network Interface**
In the network interface configuration:

- **Network interface:** Click on the dropdown and select **datacenter-nic** from the list of available network interfaces  
- Ensure the network interface is in the same region (**West US**) and VNet as the VM  
- Click **OK** or **Attach** to attach the network interface to the VM.  
![select nic](assets/day9_03.png)

**Note:** The operation may take a few moments to complete.

#### **Step 5: Verify NIC Attachment**
Once the operation completes:

- Stay on the **Network settings** page of **datacenter-vm**  
- Verify that **datacenter-nic** appears in the list of **Network interfaces**  
- The network interface should now be listed along with any existing NICs  
![verify nic attachment](assets/day9_04.png)

#### **Step 6: Start the Virtual Machine**
- Go to **Overview** page of **datacenter-vm**  
- Click **Start**  
- Wait for the VM to start successfully  
![start vm](assets/day9_05.png)

## Important Notes

- **VM must be stopped** before attaching additional network interfaces
- The network interface must be in the **same region and VNet** as the VM
- After attaching, remember to **start the VM** to use the new network interface
- You can configure the new NIC with IP addresses, NSG rules, etc., as needed
- Multiple NICs can be attached to a VM depending on the VM size