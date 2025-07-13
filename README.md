# 🖥️ AWS Lambda: EC2 Instance Summary to Slack

This AWS Lambda function scans multiple AWS regions to list all EC2 instances in `running` or `stopped` state. It then sends a formatted summary to a Slack channel via a Slack webhook.

---

## 📌 Features

- Lists EC2 instance details across multiple regions
- Filters instances by `running` and `stopped` states
- Reports the following per instance:
  - AWS Region
  - Instance State
  - Instance Type
- Sends a neat summary to Slack
- Easily schedulable with Amazon EventBridge (Cron)

---

## 🚀 Setup Instructions

### 1. Deploy the Lambda Function

Use the AWS Lambda Console or AWS CLI to create the function.

### 2. Set Environment Variable

In your Lambda configuration, add the following environment variable:

| Key               | Value                    |
|------------------|--------------------------|
| `SLACK_WEBHOOK_URL` | Paste the your Slack Webhook URL channel |

> 💡 You can create a Slack webhook [here](https://api.slack.com/messaging/webhooks).

### 3. IAM Role Permissions

Attach the following IAM permissions to your Lambda execution role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2Describe",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}

### 4. Lambda Code 




