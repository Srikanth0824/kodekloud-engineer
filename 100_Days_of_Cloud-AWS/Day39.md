## Task: Hosting a Static Website on AWS S3
The Nautilus DevOps team has been tasked with creating an internal information portal for public access. As part of this project, they need to host a static website on AWS using an S3 bucket. The S3 bucket must be configured for public access to allow external users to access the static website directly via the S3 website URL.

**Task Requirements:**
1. Create an S3 bucket named `nautilus-web-11606`.
2. Configure the S3 bucket for static website hosting with `index.html` as the index document.
3. Allow public access to the bucket so that the website is publicly accessible.
4. Upload the `index.html` file from the `/root/` directory of the AWS client host to the S3 bucket.
5. Verify that the website is accessible directly through the S3 website URL.

---

## Solution

### Step 1: Set Variables
```bash
BUCKET="nautilus-web-11606"
```

### Step 2: Create the S3 Bucket
```bash
aws s3api create-bucket \
  --bucket $BUCKET
```

### Step 3: Disable Block Public Access
```bash
aws s3api put-public-access-block \
  --bucket $BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

### Step 4: Enable Static Website Hosting
```bash
aws s3 website s3://$BUCKET/ \
  --index-document index.html
```

### Step 5: Apply Public Read Bucket Policy
Create policy file
```bash
cat > bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/*"
    }
  ]
}
EOF
```
Apply policy
```bash
aws s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy file://bucket-policy.json
```

### Step 6: Upload index.html File
```bash
# upload index.html to s3 bucket
aws s3 cp /root/index.html s3://$BUCKET/

# verify upload
aws s3 ls s3://$BUCKET/
```

### Step 7: Get S3 Website URL & Verify Access
Get exact website URL (assuming we're working with `us-east-1` region)
```bash
echo "http://$BUCKET.s3-website-us-east-1.amazonaws.com"
```
If everything is setup properly you should be able to access the website using the S3 website URL.