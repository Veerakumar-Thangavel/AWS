<img width="710" height="661" alt="AWS-lambda png" src="https://github.com/user-attachments/assets/0b72fd6f-a354-45c4-ba51-7bef8417d918" />


# 🔄 AWS Lambda - EC2 Start/Stop Automation with Slack Notification

This AWS Lambda function automatically **starts or stops EC2 instances** across multiple regions based on time and sends a **detailed summary to Slack**.

It also functions as a lightweight **EC2 inventory tool**, listing instance states and types across all regions.

---

## 📌 Features

- 🕒 Automatically **starts EC2 instances at a scheduled hour**
- 📴 Automatically **stops EC2 instances at another scheduled hour**
- 🌍 Scans **18 AWS regions**
- 📋 Collects per-instance:
  - AWS Region
  - Instance ID
  - Instance State
  - Instance Type
- 🔔 Sends **Slack notifications** with summaries and actions taken
- ⏰ Can be scheduled using **Amazon EventBridge (cron)**

---

## 🚀 Setup Instructions

### 1. Deploy the Lambda Function

Create a new Lambda function using **Python 3.x** runtime.

---

### 2. Set Environment Variables

Set the following environment variables in the Lambda configuration:

| Key                | Value                                |
|-------------------|----------------------------------------|
| `SLACK_WEBHOOK_URL` | Your Slack Incoming Webhook URL         |
| `TIMEZONE`         | (Optional) e.g., `Asia/Kolkata`         |
| `START_HOUR`       | (Optional) e.g., `9` (24-hr format)     |
| `STOP_HOUR`        | (Optional) e.g., `19` (24-hr format)    |

---

### 3. IAM Permissions for Lambda

Attach the following IAM policy to the Lambda execution role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

###  Lambda Function Code
```json
import os
import json
import boto3
import urllib3
from datetime import datetime
import pytz

http = urllib3.PoolManager()

def send_slack_message(slack_webhook_url, slack_message):
    payload = {'text': slack_message}
    encoded_data = json.dumps(payload).encode('utf-8')
    response = http.request(
        'POST',
        slack_webhook_url,
        body=encoded_data,
        headers={'Content-Type': 'application/json'}
    )
    print('>send_slack_message:response status code:', response.status)

def get_current_hour(timezone_str):
    tz = pytz.timezone(timezone_str)
    return datetime.now(tz).hour

def manage_ec2_instances(action, region, instance_ids):
    ec2 = boto3.client('ec2', region_name=region)
    if action == "start":
        ec2.start_instances(InstanceIds=instance_ids)
    elif action == "stop":
        ec2.stop_instances(InstanceIds=instance_ids)

def find_and_manage_instances():
    ec2_regions = [
        'us-east-1', 'us-east-2', 'us-west-1', 'us-west-2',
        'ap-south-1', 'ap-northeast-1', 'ap-northeast-2', 'ap-northeast-3',
        'ap-southeast-1', 'ap-southeast-2', 'ca-central-1',
        'eu-central-1', 'eu-west-1', 'eu-west-2', 'eu-west-3',
        'eu-north-1', 'sa-east-1'
    ]

    slack_webhook_url = os.environ['SLACK_WEBHOOK_URL']
    timezone = os.environ.get('TIMEZONE', 'Asia/Kolkata')
    start_hour = int(os.environ.get('START_HOUR', '9'))
    stop_hour = int(os.environ.get('STOP_HOUR', '19'))

    current_hour = get_current_hour(timezone)
    slack_message = f"*EC2 Automation Summary - Hour {current_hour} ({timezone})*\n"
    action = None

    if current_hour == start_hour:
        action = 'start'
    elif current_hour == stop_hour:
        action = 'stop'

    total_instances = 0

    for region in ec2_regions:
        ec2 = boto3.client('ec2', region_name=region)
        try:
            reservations = ec2.describe_instances(
                Filters=[{'Name': 'instance-state-name', 'Values': ['running', 'stopped']}]
            ).get('Reservations', [])

            instance_ids_to_manage = []
            for reservation in reservations:
                for instance in reservation['Instances']:
                    total_instances += 1
                    instance_id = instance['InstanceId']
                    instance_type = instance.get('InstanceType', 'N/A')
                    state = instance.get('State', {}).get('Name', 'N/A')

                    slack_message += f"• Region: `{region}` | Instance: `{instance_id}` | State: `{state}` | Type: `{instance_type}`\n"

                    if action == "start" and state == "stopped":
                        instance_ids_to_manage.append(instance_id)
                    elif action == "stop" and state == "running":
                        instance_ids_to_manage.append(instance_id)

            if instance_ids_to_manage:
                manage_ec2_instances(action, region, instance_ids_to_manage)

        except Exception as e:
            slack_message += f"• Region: `{region}` - Error: {str(e)}\n"

    if total_instances > 0:
        send_slack_message(slack_webhook_url, slack_message)

    return total_instances

def lambda_handler(event, context):
    total = find_and_manage_instances()
    return {
        'statusCode': 200,
        'body': json.dumps(f'Total EC2 Instances (Running & Stopped): {total}')
    }
```

### 5. 🕘 Scheduling Lambda with Amazon EventBridge
Step 1: Create IAM Role for Scheduler
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
Permissions Policy:
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

Step 2: Create a Cron-Based Rule
Go to EventBridge > Scheduler > Create schedule.
Name: ec2-auto-startstop
Schedule type: Cron-based
Examples:
cron(0 3 * * ? *) → triggers at 3 AM UTC daily
cron(30 13 * * ? *) → triggers at 7 PM IST (13:30 UTC)
Target: Your Lambda function
Role: Use the IAM role created above


### ✅ Example Outputs  🔔 Slack Message
*EC2 Automation Summary - Hour 9 (Asia/Kolkata)**EC2 Automation Summary - Hour 9 (Asia/Kolkata)*
• Region: `us-east-1` | Instance: `i-0abcd1234efgh5678` | State: `stopped` | Type: `t2.micro`
• Region: `us-east-1` | Instance: `i-1abcd1234efgh5679` | State: `running` | Type: `t3.large`

✅ *Started Instances:*
`i-0abcd1234efgh5678` in `us-east-1`

📊 *Total EC2 Instances (Running & Stopped):* `2`


📦 Lambda Output
```json
{
  "statusCode": 200,
  "body": "\"Total EC2 Instances (Running & Stopped): 7\""
}
```

📬 Author
Veerakumar T – DevOps Engineer
AWS | Terraform | Jenkins | Slack | EC2 Automation
