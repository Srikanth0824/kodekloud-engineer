## Task: Expanding EC2 Instance Storage for Development Needs
The Nautilus DevOps Team has recently been informed by the Development Team that their EC2 instance is running out of storage space. This instance, crucial for development activities, is named `datacenter-ec2` and currently has an attached volume of `8 GiB`. To accommodate the increasing data requirements, the storage needs to be expanded to `12 GiB`. This change should ensure that the expanded space is immediately available for use within the instance without disrupting ongoing activities.

1. **Identify Volume**: Find the volume attached to the `datacenter-ec2` instance.
2. **Expand Volume**: Increase the volume size from `8 GiB` to `12 GiB`.
3. **Reflect Changes**: Ensure the root (`/`) partition within the instance reflects the expanded size from `8 GiB` to `12 GiB`.
4. **SSH Access**: Use the key pair located at `/root/datacenter-keypair.pem` on the `aws-client` host to SSH into the EC2 instance.

---

## Solution

### Step 1: Modify the volume size
- Login to AWS management console
- Locate the `datacenter-ec2` EC2 instance and identify the volume attached to it
- Expand the volume size to `12 GiB`

### Step 2: SSH into the EC2 Instance
- Use the `/root/datacenter-keypair.pem` key on the `aws-client` host to SSH into the instance
```bash
ssh -i /root/datacenter-keypair.pem ubuntu@<EC2_IP>
```

### Step 3: Verify Current Disk Size
```bash
lsblk
df -h /
```

### Step 4: Expand the Partition and Filesystem
Grow the partition
```bash
sudo growpart /dev/xvda 1
```
Check filesystem type
```bash
df -T /
```
If `xfs`:
```bash
sudo xfs_growfs /
```
If `ext4`:
```bash
sudo resize2fs /dev/xvda1
```

### Step 5: Verification
Check if the expansion is done
```bash
df -h /
```
