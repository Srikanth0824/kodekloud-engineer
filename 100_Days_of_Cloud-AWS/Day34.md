## Task: Create a Lambda Function Using CLI
The Nautilus DevOps team continues to explore serverless architecture by setting up another Lambda function. This time, the task must be completed using the AWS Console to familiarize the team with the web interface. The function will return a custom greeting and demonstrate the capabilities of AWS Lambda effectively.

1. **Create Python Script**: Create a Python script named `lambda_function.py` with a function that returns the body `Welcome to KKE AWS Labs!` and status code `200`.
2. **Zip the Python Script**: Zip the script into a file named `function.zip`.
3. **Create Lambda Function**: Create a Lambda function named `nautilus-lambda-cli` using the zipped file and specify `Python` as the runtime.
4. **IAM Role**: Use the IAM role named `lambda_execution_role`.

---

## Solution

### Step 1: Set Variables
```bash
LAMBDA_NAME="nautilus-lambda-cli"
RUNTIME="python3.9"
IAM_ROLE="lambda_execution_role"
```

### Step 2: Create Lambda Function Code
```bash
cat > lambda_function.py <<EOF
def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "body": "Welcome to KKE AWS Labs!"
    }
EOF
```

### Step 3: Zip the function
```bash
zip function.zip lambda_function.py
```

### Step 4: Get role ARN
```bash
ROLE_ARN=$(aws iam get-role \
  --role-name $IAM_ROLE \
  --query "Role.Arn" \
  --output text)
```

### Step 5: Create the Lambda Function
```bash
aws lambda create-function \
  --function-name $LAMBDA_NAME \
  --runtime $RUNTIME \
  --role "$ROLE_ARN" \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip
```

### Step 6: Verify Lambda Function
Invoke the function
```bash
aws lambda invoke \
  --function-name $LAMBDA_NAME \
  output.json
```
Check output
```bash
cat output.json
```
![lambda output](assets/day33_01.png)