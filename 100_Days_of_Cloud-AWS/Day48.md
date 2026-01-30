## Task: Automating Infrastructure Deployment with AWS CloudFormation

The Nautilus DevOps team needs to implement a Lambda function using a CloudFormation stack. Create a CloudFormation template named `/root/nautilus-lambda.yml` on the AWS client host and configure it to create the following components. The stack name must be `nautilus-lambda-app`.
1. Create a Lambda function named `nautilus-lambda`.
2. Use the Runtime `Python`.
3. The function should print the body `Welcome to KKE AWS Labs!`.
4. Ensure the status code is `200`.
5. Create and use the IAM role named `lambda_execution_role`.

---

## Solution

### Step 1: Create the `/root/nautilus-lambda.yml` file on AWS client host:
```bash
vi /root/nautilus-lambda.yml
```
Add the following content
```yml
AWSTemplateFormatVersion: '2010-09-09'
Description: CloudFormation stack to create a basic Lambda function

Resources:

  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: lambda_execution_role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  NautilusLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: nautilus-lambda
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Timeout: 10
      Code:
        ZipFile: |
          def lambda_handler(event, context):
              return {
                  "statusCode": 200,
                  "body": "Welcome to KKE AWS Labs!"
              }
```

### Step 2: Create the stack
```bash
aws cloudformation create-stack \
  --stack-name nautilus-lambda-app \
  --template-body file:///root/nautilus-lambda.yml \
  --capabilities CAPABILITY_NAMED_IAM
```
Wait for it to be created
```bash
aws cloudformation wait stack-create-complete \
  --stack-name nautilus-lambda-app
```

### Step 3: Verification
Invoke the function
```bash
aws lambda invoke \
  --function-name nautilus-lambda \
  response.json
```
Check the output - expected output:
```json
{
  "statusCode": 200,
  "body": "Welcome to KKE AWS Labs!"
}
```
