## Task: Troubleshooting Internet Accessibility for an EC2-Hosted Application
The Nautilus Development Team recently deployed a new web application hosted on an EC2 instance within a public VPC named `devops-vpc`. The application, running on an Nginx server, should be accessible from the internet on port `80`. Despite configuring the security group `devops-sg` to allow traffic on port `80` and verifying the EC2 instance settings, the application remains inaccessible from the internet. The team suspects that the issue might be related to the VPC configuration, as all other components appear to be set up correctly. The DevOps team has been asked to troubleshoot and resolve the issue to ensure the application is accessible to external users.

As a member of the Nautilus DevOps Team, your task is to perform the following:
1. **Verify VPC Configuration:** Ensure that the VPC `devops-vpc` is properly configured to allow internet access.
2. **Ensure Accessibility:** Make sure the EC2 instance `devops-ec2` running the Nginx server is accessible from the internet on port `80`.

---

## Solution

We'll be performing this task from AWS management console.

### Step 1: Check EC2 security group rules
- Make sure that the `HTTP` traffic is allowed by the security group as mentioned in the description.
- Add inbound rule to allow `SSH` access to the instance if it isn't present(we might need it to check `nginx` server status later).

### Step 2: Check VPC settings
- Check if an `internet gateway` is persent and attached to the VPC in which the EC2 instance is present.
- Attach the `internet gateway` to `devops-vpc` if it isn't attached.
- Make sure that necessary route rule is present.

### Step 3: Check website access
- Try accessing the website using EC2 instance `public IP`.
- If you are able to see the default `nginx` page, then the issue is resolved.
- Else you would have connect to EC2 instance and further verify if the `nginx` server status.