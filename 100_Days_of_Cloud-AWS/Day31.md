## Task: Configuring a Private RDS Instance for Application Development
The Nautilus Development Team is working on a new application feature that requires a reliable and scalable database solution. To facilitate development and testing, they need a new private RDS instance. This instance will be used to store critical application data and must be provisioned using the AWS free tier to minimize costs during the initial development phase. The team has chosen MySQL as the database engine due to its compatibility with their existing systems. The DevOps team has been tasked with setting up this RDS instance, ensuring that it is correctly configured and available for use by the development team.

As a member of the Nautilus DevOps Team, your task is to perform the following:

1. **Provision a Private RDS Instance**: Create a new private RDS instance named `nautilus-rds` using a `sandbox` template, further it must be a `db.t3.micro` type instance.
2. **Engine Configuration**: Use the `MySQL` engine with version `8.4.x`.
3. **Enable Storage Autoscaling**: Enable storage autoscaling and set the threshold value to `50GB`. Keep the rest of the configurations as default.
4. **Instance Availability**: Ensure the instance is in the `available` state before submitting this task.

---

## Solution

### Step 1: Set Variables
```bash
RDS_NAME="nautilus-rds"
INSTANCE_TYPE="db.t3.micro"
ENGINE="mysql"
ENGINE_VERSION="8.4.3"
MAX_STORAGE=50
SUBNET_GROUP="db-subnet-group"
SECURITY_GROUP="rds-sg"
```

### Step 2: Get Default VPC ID
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" \
  --output text)
```

### Step 3: Get Subnets in Default VPC
RDS subnet groups must include at least 2 subnets in different AZs.
```bash
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values="$VPC_ID" \
  --query "Subnets[*].SubnetId" \
  --output text)
```

### Step 4: Create DB Subnet Group
```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name $SUBNET_GROUP \
  --db-subnet-group-description "Subnet group for RDS" \
  --subnet-ids $SUBNET_IDS
```

### Step 5: Create Security Group for RDS
```bash
RDS_SG_ID=$(aws ec2 create-security-group \
  --group-name $SECURITY_GROUP \
  --description "RDS SG" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)
```

### Step 6: Create the RDS Instance
```bash
aws rds create-db-instance \
  --db-instance-identifier $RDS_NAME \
  --db-instance-class $INSTANCE_TYPE \
  --engine $ENGINE \
  --engine-version $ENGINE_VERSION \
  --allocated-storage 20 \
  --max-allocated-storage $MAX_STORAGE \
  --master-username admin \
  --master-user-password AdminPass \
  --db-subnet-group-name $SUBNET_GROUP \
  --vpc-security-group-ids "$RDS_SG_ID" \
  --no-publicly-accessible \
  --backup-retention-period 1 \
  --storage-type gp2
```

### Step 7: Wait Until AVAILABLE
```bash
aws rds wait db-instance-available \
  --db-instance-identifier $RDS_NAME
```

### Step 8: Validate the RDS instance state
```bash
aws rds describe-db-instances \
  --db-instance-identifier $RDS_NAME \
  --query "DBInstances[0].[DBInstanceStatus,PubliclyAccessible,EngineVersion,DBInstanceClass]" \
  --output table
```
![RDS Status](assets/day31_01.png)