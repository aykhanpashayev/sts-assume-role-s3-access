# 🔐 AWS STS: Temporary S3 Access with AssumeRole
Goal: Demonstrate secure, temporary access to a specific S3 bucket using AWS STS AssumeRole.
What you’ll learn: least privilege, trust policies, short-lived creds, and safe CLI workflows.

## ✅ Prerequisites
-AWS account with IAM access (admin or equivalent)
-AWS CLI v2 configured (recommended to use a dedicated profile)
-A private bucket to test with
-Basic bash
-(Optional) CloudFormation familiarity

## 🧰 Variables (use placeholders — don’t hardcode secrets)
Store the credentials inside of hidden .aws/credentials file.
Example:
```
AWS_PROFILE=<YOUR_AWS_PROFILE>
AWS_REGION=<YOUR_REGION>              # e.g. us-east-1
ACCOUNT_ID=<123456789012>
USER_NAME=<sts-machine-user>          # IAM user that will call AssumeRole
ROLE_NAME=<sts-demo-role>             # IAM role to be assumed
BUCKET_NAME=<your-bucket-name>        # target S3 bucket
STACK_NAME=<sts-demo-stack>
```
## 🏗️ Architecture (high level)
```
IAM User (<USER_NAME>)  --AssumeRole-->  IAM Role (<ROLE_NAME>)  --> limited S3 access (s3://<BUCKET_NAME>)
                 ^                   (STS returns temporary credentials)
                 |
             AWS CLI
```
## 1) Environment Setup
### 1.1 Create a least-privilege IAM User (no permissions initially)
```sh
aws iam create-user \
  --user-name "<USER_NAME>" \
  --profile "$AWS_PROFILE"
```
### Create an access key for local testing (store it safely):
```sh
aws iam create-access-key \
  --user-name "<USER_NAME>" \
  --profile "$AWS_PROFILE"
```

### Tip: Consider using named profiles instead of raw env vars:
```
aws configure --profile <YOUR_AWS_PROFILE>
```

## 2) Deploy the Role with CloudFormation
2.1. First you will need a Template yaml or json file. I preferred yaml file you can check the file on root folder of repo.
2.2. A script will be needed to execute you can find the script: /bin/deploy

Tip: Make sure to make the deploy to executable:
```
chmoud u+x deploy
```

2.3 Deploy using the script:
```
./deploy.sh
```

Get the Role ARN from stack outputs:
```
aws cloudformation describe-stacks \
  --stack-name "$STACK_NAME" \
  --query "Stacks[0].Outputs[?OutputKey=='RoleArn'].OutputValue" \
  --output text \
  --profile "$AWS_PROFILE"
```

## 3) Allow the User to Call sts:AssumeRole
3.1. Policy needed to create check out the policy-assume-role.json file in repo's root folder.
3.2. Attach the policy:
```
ROLE_ARN="arn:aws:iam::$ACCOUNT_ID:role/$ROLE_NAME"
sed "s#<ACCOUNT_ID>#$ACCOUNT_ID#g; s#<ROLE_NAME>#$ROLE_NAME#g" policy-assume-role.json > /tmp/policy.json

aws iam put-user-policy \
  --user-name "$USER_NAME" \
  --policy-name "AllowAssumeRole-$ROLE_NAME" \
  --policy-document file:///tmp/policy.json \
  --profile "$AWS_PROFILE"
```

## 4) Assume the Role & Export Temporary Credentials
```
ASSUME_OUTPUT=$(aws sts assume-role \
  --role-arn "$ROLE_ARN" \
  --role-session-name "s3-sts-session" \
  --query "Credentials" \
  --output json \
  --profile "$AWS_PROFILE")

export AWS_ACCESS_KEY_ID=$(echo "$ASSUME_OUTPUT" | jq -r '.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo "$ASSUME_OUTPUT" | jq -r '.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo "$ASSUME_OUTPUT" | jq -r '.SessionToken')
```
Verify your identity:
```
aws sts get-caller-identity
```
## 5) Test S3 Access (scoped to your bucket)
```
aws s3 ls "s3://$BUCKET_NAME"
echo "hello" > hello.txt
aws s3 cp hello.txt "s3://$BUCKET_NAME/"
aws s3 rm "s3://$BUCKET_NAME/hello.txt"
```

## 6) Cleanup
```
aws iam delete-user-policy \
  --user-name "$USER_NAME" \
  --policy-name "AllowAssumeRole-$ROLE_NAME" \
  --profile "$AWS_PROFILE"

aws cloudformation delete-stack \
  --stack-name "$STACK_NAME" \
  --profile "$AWS_PROFILE"

aws iam delete-user --user-name "$USER_NAME" --profile "$AWS_PROFILE"
```

## 🔒 Security Notes
-Never commit keys, tokens, or raw ARNs tied to your real identity.
-Use placeholders like <ACCOUNT_ID>, <USER_NAME>, <ROLE_NAME>, <BUCKET_NAME>.
-Prefer AWS named profiles over exporting long-lived access keys.
-Limit S3 actions to exactly what you need (List/Get/Put/Delete in this demo).
-Use short session durations in production roles.

## 🛠️ Troubleshooting (common real-world errors)
AccessDenied on S3 list:
```
Ensure the role policy includes s3:ListBucket for the bucket ARN (no /*) and your CLI is using temporary creds.
```
MalformedPolicyDocument:
```
Validate JSON; for trust policy, Principal under role is the user ARN; for inline user policy, Resource is the role ARN.
```
InvalidToken:
```
Your session token expired or you forgot to export AWS_SESSION_TOKEN. Re-assume the role.
```

## 🧠 What This Demonstrates
-Temporary credentials via STS (reduced blast radius)
-Principle of least privilege (bucket-scoped)
-Infra as Code with CloudFormation
-Clean, reproducible security demo with safe placeholders
