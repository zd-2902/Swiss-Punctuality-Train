# Swiss-Punctuality-Train

##  Get started

### Base Setup

1. Install AWS CLI
2. Start your AWS lab environment
3. Click on `AWS Details`
4. Click on `Show` next to `AWS CLI`
5. Copy the shown content and add it to your local `~/.aws/credentials` file
6. Run `aws s3 ls` to verify that you have access to the S3 bucket
7. Configure the default region with `aws configure set region us-east-1`
8. Configure the default output format with `aws configure set output json`
9. Export the AWS account ID as an environment variable:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
export LAB_ROLE_ARN=$(aws iam get-role --role-name LabRole --query 'Role.Arn')
```

### CloudFormation bucket

To deploy the CloudFormation stack, you need to create a bucket to which we can upload the CloudFormation template, lambda layers and lambda functions.

1. Create a new cloudformation stack with the following command:

```bash
aws cloudformation create-stack \
    --stack-name cf-deployment-bucket \
    --template-body file://cf/1_bucket.yaml
```

2. Wait until the stack is created
3. Run `aws s3 ls s3://$AWS_ACCOUNT_ID-cf-deployment-bucket` to verify that the bucket has been created. When no error is shown, the bucket has been created successfully.

### Build and deploy the lambda functions

1. Run the following command to build and deploy the lambda functions:

```bash
./layers.sh
```

### Deploy the CloudFormation stack

1. Get the ARN of the `LabRole` role by running the following command:

```bash
aws iam get-role \
    --role-name LabRole \
    --query 'Role.Arn'
```

2. Export the ARN as an environment variable:

```bash
export LAB_ROLE_ARN=<arn>
```

3. Run the following command to deploy the CloudFormation stack:

```bash
aws cloudformation deploy \
  --stack-name hslu-dwh \
  --template-file cf/2_dwh.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides RoleArn=$LAB_ROLE_ARN CfBucketName=$AWS_ACCOUNT_ID-cf-deployment-bucket
```

4. Update the `hslu-dwh` stack with the following command:

```bash
LAMBDA_ARN=$(aws lambda get-function --function-name ETLPubTransData --query 'Configuration.FunctionArn' --output text)
BUCKET_NAME="${AWS_ACCOUNT_ID}-data-lake-bucket"
aws s3api put-bucket-notification-configuration \
  --bucket "$BUCKET_NAME" \
  --notification-configuration '{
      "LambdaFunctionConfigurations": [
        {
          "LambdaFunctionArn": "'"$LAMBDA_ARN"'",
          "Events": ["s3:ObjectCreated:Put"],
          "Filter": {
            "Key": {
              "FilterRules": [
                {
                  "Name": "prefix",
                  "Value": "raw/pubtrans"
                }
              ]
            }
          }
        }
      ]
    }'
```

## Usage

TODO

##  Cleanup

1. Clear the S3 buckets:

```bash
aws s3 rm s3://$AWS_ACCOUNT_ID-cf-deployment-bucket --recursive
aws s3 rm s3://$AWS_ACCOUNT_ID-data-lake-bucket --recursive
```

1. Delete the CloudFormation stack:

```bash
aws cloudformation delete-stack --stack-name hslu-dwh
aws cloudformation delete-stack --stack-name cf-deployment-bucket
```

2. Stop the lab environment
