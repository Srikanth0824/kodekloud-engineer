## Task: Centralized Audit Logging with VPC Peering
The Nautilus DevOps team needs to build a secure and scalable log aggregation setup within their AWS environment. The goal is to gather log files from an internal EC2 instance running in a private VPC, transfer them securely to another EC2 instance in a public VPC, and then push those logs to a secure S3 bucket.

1. A VPC named `nautilus-priv-vpc` already exists with a private subnet named `nautilus-priv-subnet`, a route table named `nautilus-priv-rt`, and an EC2 instance named `nautilus-priv-ec2` (using `ubuntu` image). This instance uses the SSH key pair `nautilus-key.pem` already available on the AWS client host at `/root/.ssh/`.
2. Your task is to:
    - Create a new VPC named `nautilus-pub-vpc`.
    - Create a subnet named `nautilus-pub-subnet` and a route table named `nautilus-pub-rt` under this public VPC.
    - Attach an internet gateway to `nautilus-pub-vpc` and configure the public route table to enable internet access.
    - Launch an EC2 instance named `nautilus-pub-ec2` into the public subnet using the same key pair as the private instance.
    - Create an IAM role named `nautilus-s3-role` with `PutObject` permission to an S3 bucket and attach it to the public EC2 instance.
    - Create a new private S3 bucket named `nautilus-s3-logs-27334`.
    - Configure a VPC Peering named `nautilus-vpc-peering` between the private and public VPCs.
    - Modify both `nautilus-priv-rt` and `nautilus-pub-rt` to route each other's CIDR blocks through the peering connection.
    - On the private instance, configure a cron job to push the `/var/log/boots.log` file to the public instance (using `scp` or `rsync`).
    - On the public instance, configure a cron job to push that same file to the created S3 bucket.
    - The uploaded file must be stored in the S3 bucket under the path `nautilus-priv-vpc/boot/boots.log`.

---

## Solution

### Step 1: Set Variables
```bash
PRIV_VPC="nautilus-priv-vpc"
PRIV_RT="nautilus-priv-rt"
PUB_VPC="nautilus-pub-vpc"
PUB_SUBNET="nautilus-pub-subnet"
PUB_RT="nautilus-pub-rt"
VPC_PEERING="nautilus-vpc-peering"
PUB_EC2="nautilus-pub-ec2"
PRIV_S3="nautilus-s3-logs-27334"
S3_ROLE="nautilus-s3-role"
```

### Step 2: Create network components
Create VPC
```bash
PUB_VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.20.0.0/16 \
  --query "Vpc.VpcId" \
  --output text)

aws ec2 create-tags \
  --resources $PUB_VPC_ID \
  --tags Key=Name,Value=$PUB_VPC
```
Create Subnet
```bash
PUB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $PUB_VPC_ID \
  --cidr-block 10.20.1.0/24 \
  --query "Subnet.SubnetId" \
  --output text)

aws ec2 create-tags \
  --resources $PUB_SUBNET_ID \
  --tags Key=Name,Value=$PUB_SUBNET
```
Enable auto PublicIP
```bash
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_ID \
  --map-public-ip-on-launch
```
Internet gateway
```bash
# Create IG
IGW_ID=$(aws ec2 create-internet-gateway \
  --query "InternetGateway.InternetGatewayId" \
  --output text)
# Attach IG
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $PUB_VPC_ID
```
Create Route table
```bash
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $PUB_VPC_ID \
  --query "RouteTable.RouteTableId" \
  --output text)

aws ec2 create-tags \
  --resources $PUB_RT_ID \
  --tags Key=Name,Value=$PUB_RT
```
Add internet route
```bash
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID
```
Associate subnet
```bash
aws ec2 associate-route-table \
  --subnet-id $PUB_SUBNET_ID \
  --route-table-id $PUB_RT_ID
```

### Step 3: Create EC2 instance
```bash
# get AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*" \
  --query "Images[0].ImageId" \
  --output text)

# Create EC2 instance
PUB_EC2_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name nautilus-key \
  --subnet-id $PUB_SUBNET_ID \
  --query "Instances[0].InstanceId" \
  --output text)

# Tag EC2 instance
aws ec2 create-tags \
  --resources $PUB_EC2_ID \
  --tags Key=Name,Value=$PUB_EC2
```

### Step 4: Create Private S3 bucket
```bash
aws s3api create-bucket \
  --bucket $PRIV_S3 \
  --region us-east-1
```

### Step 5: Create IAM Role for Public EC2
Create trust policy file
```bash
cat > trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF
```
Create role
```bash
aws iam create-role \
  --role-name $S3_ROLE \
  --assume-role-policy-document file://trust.json
```
Create permission policy file
```bash
cat > s3-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::$PRIV_S3/*"
  }]
}
EOF
```
```bash
aws iam create-policy \
  --policy-name s3-put-policy \
  --policy-document file://s3-policy.json
```
```bash
aws iam attach-role-policy \
  --role-name $S3_ROLE \
  --policy-arn arn:aws:iam::480283352309:policy/s3-put-policy
```
Instance profile
```bash
aws iam create-instance-profile \
  --instance-profile-name $S3_ROLE
```
```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name $S3_ROLE \
  --role-name $S3_ROLE
```
Attach to EC2 instance
```bash
aws ec2 associate-iam-instance-profile \
  --instance-id $PUB_EC2_ID \
  --iam-instance-profile Name=$S3_ROLE
```

### Step 6: Configure VPC Peering
```bash
PRIV_VPC_ID=$(aws ec2 describe-vpcs \
  --filters Name=tag:Name,Values=$PRIV_VPC \
  --query "Vpcs[0].VpcId" \
  --output text)
```
```bash
PEER_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $PRIV_VPC_ID \
  --peer-vpc-id $PUB_VPC_ID \
  --query "VpcPeeringConnection.VpcPeeringConnectionId" \
  --output text)
```
Accept peering connection
```bash
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEER_ID
```
Update peering connection name
```bash
aws ec2 create-tags \
  --resources $PEER_ID \
  --tags Key=Name,Value=$VPC_PEERING
```

### Step 7: Update Route tables
Get CIDRs
```bash
PRIV_CIDR=$(aws ec2 describe-vpcs --vpc-ids $PRIV_VPC_ID --query "Vpcs[0].CidrBlock" --output text)
PUB_CIDR=$(aws ec2 describe-vpcs --vpc-ids $PUB_VPC_ID --query "Vpcs[0].CidrBlock" --output text)
```
Private RT
```bash
PRIV_RT_ID=$(aws ec2 describe-route-tables \
  --filters Name=tag:Name,Values=$PRIV_RT \
  --query "RouteTables[0].RouteTableId" \
  --output text)

aws ec2 create-route \
  --route-table-id $PRIV_RT_ID \
  --destination-cidr-block $PUB_CIDR \
  --vpc-peering-connection-id $PEER_ID
```
Public RT
```bash
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block $PRIV_CIDR \
  --vpc-peering-connection-id $PEER_ID
```

### Step 8: Configure Cron job on public EC2 instance
- Connect to public EC2 instance from the `aws-client` host 
- Update the crontab to upload logs to S3 bucket
```bash
crontab -e
```
```bash
* * * * * aws s3 cp /root/boots.log s3://nautilus-s3-logs-27334/nautilus-priv-vpc/boot/boots.log
```
- Ensure awscli is installed on the public EC2 instance

### Step 9: Configure Cron job on private EC2 instance
- Connect to private EC2 instance from the public instance using private IP
- Update crontab to upload logs to public instance 
```bash
* * * * * scp -i /root/nautilus-key.pem /var/log/boots.log root@<pub_instance_private_IP>:/root/boots.log
```
- Ensure private instance has the ssh key(`nautilus-key.pem`) to connect to the public instance.

### Step 10: Verification
Check if the log file is uploaded to the private S3 bucket 