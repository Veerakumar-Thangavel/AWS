# 🖥️ AWS Lambda: EC2 Instance Summary to Slack Notification

This AWS Lambda function scans multiple AWS regions to list all EC2 instances in `running` or `stopped` state. It then sends a formatted summary to a Slack channel via a Slack webhook.

---
## Architecture Diagram

<img width="710" height="661" alt="AWS-lambda png" src="https://github.com/user-attachments/assets/0b72fd6f-a354-45c4-ba51-7bef8417d918" />

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
```
### 4. AWS Lambd Code 

    import os
    import json
    import boto3
    import urllib3

    http = urllib3.PoolManager()

    def send_slack_message(slack_webhook_url, slack_message):
    payload = {
        'text': slack_message
    }

    encoded_data = json.dumps(payload).encode('utf-8')
    response = http.request(
        'POST',
        slack_webhook_url,
        body=encoded_data,
        headers={'Content-Type': 'application/json'}
    )
    print('>send_slack_message:response status code:', response.status)

    def find_ec2_instance_summary():
    ec2_regions = [
        'us-east-1', 'us-east-2', 'us-west-1', 'us-west-2',
        'ap-south-1', 'ap-northeast-1', 'ap-northeast-2', 'ap-northeast-3',
        'ap-southeast-1', 'ap-southeast-2', 'ca-central-1',
        'eu-central-1', 'eu-west-1', 'eu-west-2', 'eu-west-3',
        'eu-north-1', 'sa-east-1'
    ]

    slack_webhook_url = os.environ['SLACK_WEBHOOK_URL']
    message = '* EC2 Summary: Region, State & Type*\n'
    total_instances = 0

    for region in ec2_regions:
        ec2 = boto3.client('ec2', region_name=region)
        try:
            reservations = ec2.describe_instances(
                Filters=[
                    {'Name': 'instance-state-name', 'Values': ['running', 'stopped']}
                ]
            ).get('Reservations', [])

            for reservation in reservations:
                for instance in reservation['Instances']:
                    total_instances += 1
                    instance_type = instance.get('InstanceType', 'N/A')
                    state = instance.get('State', {}).get('Name', 'N/A')

                    message += f"• Region: `{region}` | State: `{state}` | Type: `{instance_type}`\n"

        except Exception as e:
            message += f"• Region: `{region}` - Error: {str(e)}\n"

    if total_instances > 0:
        send_slack_message(slack_webhook_url, message)

    return total_instances

    def lambda_handler(event, context):
    total = find_ec2_instance_summary()
    return {
        'statusCode': 200,
        'body': json.dumps(f'Total EC2 Instances (Running & Stopped): {total}')
    }

Explaination about the lambda code 

This AWS Lambda function provides a centralized summary of EC2 instances across all AWS regions. It scans for EC2 instances that are in either a running or stopped state and extracts key details, including:
  - AWS Region where the instance resides
  - Instance State (e.g., running, stopped)
  - Instance Type (e.g., t2.micro, m5.large)
  - Cross-region visibility into all EC2 instances in your AWS account
  - A lightweight inventory report directly sent to your Slack channel
  - A fast way to identify resource usage, idle instances, or regional distribution
  - An easy integration point for daily monitoring via scheduled triggers (e.g., EventBridge)

### 5.  AWS EventBridge Scheduler Setup to Trigger Lambda

##  Create an IAM Role for EventBridge Scheduler

Create an IAM role that allows EventBridge Scheduler to invoke the target Lambda function.

### Trust Policy (Assume Role)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "scheduler.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
Permissions Policy (Attach to Role)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:<region>:<account-id>:function:<lambda-function-name>"
    }
  ]
}
```
Create a Rule in EventBridge Scheduler 

- Open Amazon EventBridge in the AWS Console.
- Go to Scheduler > Create schedule.
- Enter a Name for the schedule (e.g., daily-lambda-trigger).
- Choose Frequency: Select Cron-based schedule.
- Set your Cron expression (e.g., cron(0 3 * * ? *) for 3 AM daily).
- In the Target section:
    Choose Lambda function.
    Target Lambda.
    Assign the IAM role created in Step 1 as the Execution role.

### Final OutPut will be recived through Slack
Slack Meaasage Output 
```json
EC2 Summary: Region, State & Type
• Region: `us-east-1` | State: `running` | Type: `t3.micro`
• Region: `us-east-1` | State: `stopped` | Type: `t2.medium`
• Region: `eu-west-1` | State: `running` | Type: `m5.large`
• Region: `ap-south-1` - Error: Auth failure or region not enabled
```
Lambda function console Output 
```json
{
  "statusCode": 200,
  "body": "\"Total EC2 Instances (Running & Stopped): 12\""
}
```



