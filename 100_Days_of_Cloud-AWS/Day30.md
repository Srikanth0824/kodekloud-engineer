## Task: Enable Internet Access for Private EC2 using NAT Instance
The Nautilus DevOps team is tasked with enabling internet access for an EC2 instance running in a private subnet. This instance should be able to upload a test file to a public S3 bucket once it can access the internet. To minimize costs, the team has decided to use a NAT Instance instead of a NAT Gateway.

The following components already exist in the environment:
1. A VPC named `xfusion-priv-vpc` and a private subnet named `xfusion-priv-subnet` have been created.
2. An EC2 instance named `xfusion-priv-ec2` is already running in the private subnet.
3. The EC2 instance is configured with a cron job that uploads a test file to the S3 bucket `xfusion-nat-31533` every minute. Upload will only succeed once internet access is established.

Your task is to:
- Create a new public subnet named `xfusion-pub-subnet` in the existing VPC.
- Launch a NAT Instance in the public subnet using an Amazon Linux 2023 AMI and name it `xfusion-nat-instance`. Configure this instance to act as a NAT instance. Make sure to use a custom security group for this instance.

After the configuration, verify that the test file `xfusion-test.txt` appears in the S3 bucket `xfusion-nat-31533`. This indicates successful internet access from the private EC2 instance via the NAT Instance.

**Note**: `iptables` is not installed by default on `Amazon Linux 2023`. You will need to install and enable it before configuring NAT setup.

---

## Solution

### Step 1: Set Variables

Be sure to read the question carefully and adjust the values below to match before running this.
```bash
VPC_NAME="xfusion-priv-vpc"
PRIV_SUBNET_NAME="xfusion-priv-subnet"
PRIV_EC2_NAME="xfusion-priv-ec2"
PUB_SUBNET_NAME="xfusion-pub-subnet"
NAT_INSTANCE_NAME="xfusion-nat-instance"
S3_BUCKET="xfusion-nat-31533"
REGION="us-east-1"
```

### Step 2: Get VPC and Private Subnet IDs
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters Name=tag:Name,Values="$VPC_NAME" \
  --query "Vpcs[0].VpcId" \
  --output text)

PRIV_SUBNET_ID=$(aws ec2 describe-subnets \
  --filters Name=tag:Name,Values="$PRIV_SUBNET_NAME" \
  --query "Subnets[0].SubnetId" \
  --output text)
```
Get the VPC and subnet CIDRs
```bash
VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --query "Vpcs[0].CidrBlock" --output text)
PRIV_SUBNET_CIDR=$(aws ec2 describe-subnets --subnet-ids "$PRIV_SUBNET_ID" --query "Subnets[0].CidrBlock" --output text)
echo "##### Network CIDR details #####"
echo "VPC CIDR: $VPC_CIDR"
echo "Private Subnet CIDR: $PRIV_SUBNET_CIDR"
echo "################################"

if [ "$VPC_ID" == "None" ] ; then
    echo "Your variables are not set correctly (e.g. devops/xfusion/datacenter)"
fi
```

### Step 3: Create a Public Subnet
Choose a CIDR not overlapping private subnet
```bash
PUB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.1.2.0/24 \
  --query "Subnet.SubnetId" \
  --output text)

aws ec2 create-tags \
  --resources "$PUB_SUBNET_ID" \
  --tags Key=Name,Value="$PUB_SUBNET_NAME"
```
Enable auto-assign public IP
```bash
aws ec2 modify-subnet-attribute \
  --subnet-id "$PUB_SUBNET_ID" \
  --map-public-ip-on-launch
```

### Step 4: Ensure Internet Gateway Exists & Attached
```bash
IGW_ID=$(aws ec2 describe-internet-gateways \
  --filters Name=attachment.vpc-id,Values="$VPC_ID" \
  --query "InternetGateways[0].InternetGatewayId" \
  --output text)
echo "Internet Gateway ID: $IGW_ID"
```
If empty, create & attach
```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query "InternetGateway.InternetGatewayId" \
  --output text)

aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"
```

### Step 5: Create Route Table for Public Subnet
```bash
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --query "RouteTable.RouteTableId" \
  --output text)

aws ec2 create-tags \
  --resources "$PUB_RT_ID" \
  --tags Key=Name,Value=pub-rt
```
Add internet route
```bash
aws ec2 create-route \
  --route-table-id "$PUB_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID"
```
Associate with public subnet
```bash
aws ec2 associate-route-table \
  --route-table-id "$PUB_RT_ID" \
  --subnet-id "$PUB_SUBNET_ID"
```

### Step 6: Create Security Group for NAT Instance
```bash
NAT_SG_ID=$(aws ec2 create-security-group \
  --group-name nat-sg \
  --description "Security group for NAT instance" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)
```
Allow traffic from private subnet
```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$NAT_SG_ID" \
  --protocol -1 \
  --cidr "$VPC_CIDR"
```
Enable EC2 instance connect in the security group so we can connect to the instance from the console
```bash
INSTANCE_CONNECT_CIDR=$(curl -s https://ip-ranges.amazonaws.com/ip-ranges.json \
| jq -r --arg REGION "$REGION" '
  .prefixes[]
  | select(.region == $REGION and .service == "EC2_INSTANCE_CONNECT")
  | .ip_prefix
')

aws ec2 authorize-security-group-ingress \
  --group-id "$NAT_SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr "$INSTANCE_CONNECT_CIDR"
```

### Step 7: Launch NAT Instance (Amazon Linux 2023)
Get AMI ID
```bash
AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query "Parameters[0].Value" \
  --output text)
```
Script to configure Linux instance to act as a NAT instance
```bash
USER_DATA='#!/bin/bash
# Enable IP forwarding
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p

# Install iptables
yum install -y iptables-services

# Discover interface used to talk to the outside world
# In AL2023 it can be different every time!
IFACE=$(ip -o route get 1.1.1.1 | awk '\''{for(i=1;i<=NF;i++) if ($i=="dev") print $(i+1)}'\'')

# Configure iptables for NAT
iptables -t nat -A POSTROUTING -o "$IFACE" -j MASQUERADE
iptables -A FORWARD -i "$IFACE" -o "$IFACE" -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i "$IFACE" -o "$IFACE" -j ACCEPT

# Save iptables rules
iptables-save > /etc/sysconfig/iptables
'
```
Launch NAT instance
```bash
NAT_INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --subnet-id "$PUB_SUBNET_ID" \
  --security-group-ids "$NAT_SG_ID" \
  --user-data "$USER_DATA" \
  --associate-public-ip-address \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$NAT_INSTANCE_NAME}]" \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Wait for instance to start"
aws ec2 wait instance-running --instance-ids ${NAT_INSTANCE_ID}
```

### Step 8: Disable Source/Destination Check (CRITICAL)
**Note:** NAT instances must have source/destination checking disabled.
```bash
aws ec2 modify-instance-attribute \
  --instance-id "$NAT_INSTANCE_ID" \
  --no-source-dest-check
```

### Step 9: Update Private Subnet Route Table
Get private route table ID
```bash
PRIV_RT_ID=$(aws ec2 describe-route-tables \
  --filters Name=association.subnet-id,Values="$PRIV_SUBNET_ID" \
  --query "RouteTables[0].RouteTableId" \
  --output text)
```
Add NAT route
```bash
aws ec2 create-route \
  --route-table-id "$PRIV_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --instance-id "$NAT_INSTANCE_ID"
```

### Step 10: Verification
Check if the `xfusion-test.txt` file got uploaded to S3 bucket. It may take a minute or two to appear.
```bash
aws s3 ls s3://$S3_BUCKET/
```

### Additional info
**NAT configuration script** passed as User Data has the following **Three Main Components**
1. **IP Forwarding** - Allows the Linux instance to route packets (not just be an endpoint)
    - Think of your NAT instance as a post office:
      - **Without IP forwarding**: The post office only accepts mail addressed to itself
      - **With IP forwarding**: The post office accepts mail and forwards it to other addresses
2. **Firewall Rules** - Controls which traffic is allowed to pass through
    - `iptables` is Linux's built-in firewall and packet filtering system. It processes network packets through different **tables** and **chains**.
    - **Tables:**
      - `nat` table: Used for Network Address Translation
      - `filter` table: Default table for allowing/blocking traffic
    - **Chains:**
      - `POSTROUTING`: Processes packets after routing decisions (as they exit)
      - `FORWARD`: Processes packets being routed through the instance
      - `PREROUTING`: Processes packets before routing decisions (as they enter)
3. **NAT/MASQUERADE** - Replaces private source IPs with the NAT instance's public IP so internet servers can respond
    ```
    BEFORE NAT:
    Private EC2 (10.0.0.10) → NAT Instance → Internet
    Source IP: 10.0.0.10 (private, non-routable)
    Destination IP: 142.250.185.46 (google.com)

    AFTER MASQUERADE:
    Private EC2 → NAT Instance (54.123.45.67) → Internet
    Source IP: 54.123.45.67 (NAT's public IP)
    Destination IP: 142.250.185.46 (google.com)
    ```