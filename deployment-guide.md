# 🚀 AWS Serverless Feedback & Contact Form — Deployment Guide

This guide explains how to deploy the project step by step using the AWS Console.

---

## Architecture

```text
User
 ↓
CloudFront
 ↓
S3
 ↓
API Gateway
 ↓
Lambda
 ├──→ DynamoDB
 └──→ SNS
       ↓
     Email
```

---

# Step 1 — Create DynamoDB Table

1. Open **AWS Console → DynamoDB**
2. Click **Create table**
3. Configure:

```text
Table name: feedback-messages
Partition key: message_id
Type: String
Capacity mode: On-demand
```

4. Click **Create table**
5. Wait until the table status becomes **Active**

---

# Step 2 — Create SNS Topic

1. Open **Amazon SNS**
2. Go to **Topics**
3. Click **Create topic**
4. Select **Standard**
5. Topic name:

```text
feedback-notifications
```

6. Create the topic
7. Copy the Topic ARN

Example:

```text
arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:feedback-notifications
```

## Subscribe an Email

1. Open the topic
2. Click **Create subscription**
3. Protocol: **Email**
4. Enter your email address
5. Create the subscription
6. Check your inbox
7. Click **Confirm subscription**

The subscription should show **Confirmed**.

---

# Step 3 — Create IAM Policy

Go to:

**IAM → Policies → Create policy → JSON**

Create:

```text
feedback-lambda-policy
```

Use:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/feedback-messages"
    },
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "*"
    }
  ]
}
```

## Create IAM Role

1. Go to **IAM → Roles**
2. Click **Create role**
3. Trusted entity: **AWS Service**
4. Use case: **Lambda**
5. Attach:

```text
feedback-lambda-policy
```

6. Role name:

```text
feedback-lambda-role
```

7. Create the role.

---

# Step 4 — Create Lambda Function

1. Open **AWS Lambda**
2. Click **Create function**
3. Select **Author from scratch**
4. Configure:

```text
Function name: feedback-handler
Runtime: Python 3.12
```

5. Choose **Use an existing role**
6. Select:

```text
feedback-lambda-role
```

7. Create the function.

## Upload Code

Open:

```text
lambda/lambda_function.py
```

Copy the code into the Lambda editor.

Click **Deploy**.

## Environment Variables

Go to:

**Configuration → Environment variables**

Add:

| Key | Value |
|---|---|
| `TABLE_NAME` | `feedback-messages` |
| `TOPIC_ARN` | Your SNS Topic ARN |

## Timeout

Go to:

**Configuration → General configuration**

Set:

```text
Timeout: 15 seconds
```

---

# Step 5 — Create API Gateway

1. Open **API Gateway**
2. Click **Create API**
3. Select **REST API**
4. API name:

```text
feedback-api
```

5. Create the API.

## Create POST `/feedback`

1. Create resource:

```text
/feedback
```

2. Create method:

```text
POST
```

3. Integration type:

```text
Lambda Function
```

4. Enable:

```text
Use Lambda Proxy integration
```

5. Select:

```text
feedback-handler
```

6. Enable CORS.

## Create GET `/stats`

1. Create resource:

```text
/stats
```

2. Create method:

```text
GET
```

3. Integration type:

```text
Lambda Function
```

4. Enable:

```text
Use Lambda Proxy integration
```

5. Select:

```text
feedback-handler
```

6. Enable CORS.

## Deploy API

Create a stage:

```text
prod
```

The Invoke URL will look like:

```text
https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod
```

Save this URL.

---

# Step 6 — Configure Frontend

Open:

```text
frontend/script.js
```

Find:

```javascript
const API_URL = "https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/prod";
```

Replace it with your actual API Gateway Invoke URL:

```javascript
const API_URL = "https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod";
```

Save the file.

---

# Step 7 — Create S3 Bucket

1. Open **Amazon S3**
2. Click **Create bucket**
3. Use a globally unique bucket name:

```text
feedback-frontend-YOUR_ACCOUNT_ID
```

4. Select your AWS region
5. Configure public access according to the hosting approach used by the project
6. Create the bucket.

## Upload Frontend

Upload:

```text
frontend/index.html
frontend/style.css
frontend/script.js
```

## Static Website Hosting

Go to:

**Properties → Static website hosting**

Enable hosting and set:

```text
Index document: index.html
```

## Bucket Policy

If using the public S3 website configuration, go to:

**Permissions → Bucket policy**

Use:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

Replace `YOUR-BUCKET-NAME` with the actual bucket name.

---

# Step 8 — Create CloudFront Distribution

1. Open **Amazon CloudFront**
2. Click **Create distribution**
3. Select the S3 bucket as the origin
4. Configure the origin according to the S3 hosting setup
5. Set viewer protocol policy to:

```text
Redirect HTTP to HTTPS
```

6. Default root object:

```text
index.html
```

7. Create the distribution.

Wait until the distribution becomes **Enabled**.

Copy the CloudFront domain, for example:

```text
https://dxxxxxxxxxx.cloudfront.net
```

---

# Step 9 — Test the Application

## Test Lambda

Use a test event:

```json
{
  "httpMethod": "POST",
  "path": "/feedback",
  "body": "{"name":"Test User","email":"test@example.com","subject":"Hello","category":"general","message":"This is a test message."}"
}
```

Expected successful response:

```json
{
  "statusCode": 200,
  "body": "{"message_id":"uuid-here","message":"Your message has been sent successfully!"}"
}
```

## End-to-End Test

| Test | Expected Result |
|---|---|
| Open CloudFront URL | Form loads |
| Submit empty form | Validation error |
| Submit valid form | Success message |
| Check email | SNS notification received |
| Check DynamoDB | Message stored |
| Check counter | Total count updated |

---

# Step 10 — Cleanup

To remove the resources after testing:

1. Disable and delete **CloudFront**
2. Empty and delete the **S3 bucket**
3. Delete **API Gateway**
4. Delete **Lambda**
5. Delete **DynamoDB table**
6. Delete **SNS subscription and topic**
7. Delete **IAM role**
8. Delete **IAM policy**

---

## Notes

- Replace all placeholder values such as `YOUR_ACCOUNT_ID`, `YOUR-BUCKET-NAME`, and the API URL with your actual values.
- Never commit AWS access keys, secret keys, passwords, or other credentials to GitHub.
- AWS pricing and Free Tier eligibility can change; check the AWS Billing Dashboard for current usage.
