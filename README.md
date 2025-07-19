# 🔄 EC2 Automation with Slack Notification

Automate starting and stopping EC2 instances across all AWS regions based on a defined schedule using AWS Lambda and EventBridge. Receive detailed Slack notifications for full visibility.

It also functions as a lightweight **EC2 inventory tool**, listing instance states and types across all regions.

---
<img width="710" height="661" alt="AWS-lambda png" src="https://github.com/user-attachments/assets/0b72fd6f-a354-45c4-ba51-7bef8417d918" />

## 📦 Features

- 🔍 Lists all EC2 instances in **all AWS regions**
- 🚀 Starts or 🔻 stops instances based on current state and action
- 🕒 Uses **Asia/Kolkata** (IST) timezone for scheduling and reporting
- 📣 Sends formatted Slack summary with:
  - Region, Instance ID, State, Type
  - ✅ Started Instances
  - 🔻 Stopped Instances
  - 📊 Total instance count
---

## 🧠 How It Works

- You set `ACTION=start` or `ACTION=stop` as an environment variable.
- The function runs based on an **EventBridge cron schedule**.
- EC2 instances are started or stopped based on their current state.
- A **Slack message** is sent summarizing the operation.

## 🚀 Setup Instructions

### 1. Deploy the Lambda Function

• Runtime: Python 3.x

• Handler: lambda_function.lambda_handler
---

## 🛠️ Environment Variables

| Variable        | Value            | Description                         |
|----------------|------------------|-------------------------------------|
| `ACTION`        | `start` or `stop`| What action to perform              |
| `SLACK_WEBHOOK` | `<Webhook URL>`  | Slack Incoming Webhook URL          |

## ⏰ Scheduling with EventBridge (Asia/Kolkata)

Example cron schedule to run every day at 9 AM IST (UTC+5:30):

```cron
cron(30 3 * * ? *)  # 9:00 AM IST = 3:30 AM UTC
```
To stop at 7 PM IST:
```cron
cron(30 13 * * ? *)  # 7:00 PM IST = 1:30 PM UTC
```
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

```
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
    print(f"> Slack response: {response.status}")

def manage_ec2_instances(action, region, instance_ids):
    ec2 = boto3.client('ec2', region_name=region)
    if action == "start":
        ec2.start_instances(InstanceIds=instance_ids)
    elif action == "stop":
        ec2.stop_instances(InstanceIds=instance_ids)

def find_and_manage_instances():
    action = os.getenv("ACTION", "stop").lower()
    slack_webhook_url = os.getenv("SLACK_WEBHOOK")

    ec2_regions = [r['RegionName'] for r in boto3.client('ec2').describe_regions()['Regions']]
    ist_time = datetime.now(pytz.timezone('Asia/Kolkata')).strftime("%H:%M")
    slack_message = f"*EC2 Automation Summary - Hour {ist_time} (Asia/Kolkata)*\n"
    total_instances = 0
    started_instances = []
    stopped_instances = []

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
                        started_instances.append(f"`{instance_id}` in `{region}`")
                    elif action == "stop" and state == "running":
                        instance_ids_to_manage.append(instance_id)
                        stopped_instances.append(f"`{instance_id}` in `{region}`")

            if instance_ids_to_manage:
                manage_ec2_instances(action, region, instance_ids_to_manage)

        except Exception as e:
            slack_message += f"• Region: `{region}` - Error: {str(e)}\n"

    if action == "start" and started_instances:
        slack_message += "\n✅ *Started Instances:*\n" + "\n".join(started_instances)
    elif action == "stop" and stopped_instances:
        slack_message += "\n🔻 *Stopped Instances:*\n" + "\n".join(stopped_instances)

    slack_message += f"\n📊 *Total EC2 Instances (Running & Stopped):* `{total_instances}`"

    if total_instances > 0:
        send_slack_message(slack_webhook_url, slack_message)

def lambda_handler(event, context):
    find_and_manage_instances()

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
---
Step 2: Create a Cron-Based Rule

Go to EventBridge > Scheduler > Create schedule.

Name: ec2-auto-startstop

Schedule type: Cron-based

Examples:
cron(0 3 * * ? *) → triggers at 3 AM UTC daily
cron(30 13 * * ? *) → triggers at 7 PM IST (13:30 UTC)
Target: Your Lambda function
Role: Use the IAM role created above


*EC2 Automation Summary - Hour 09:00 (Asia/Kolkata)*

• Region: `us-east-1` | Instance: `i-0abc123456789xyz0` | State: `stopped` | Type: `t2.micro`

• Region: `us-west-2` | Instance: `i-0def987654321xyz1` | State: `running` | Type: `t3.medium`

✅ *Started Instances:*
`i-0abc123456789xyz0` in `us-east-1`

📊 *Total EC2 Instances (Running & Stopped):* `2`


📦 Lambda Output
```json
{
  "statusCode": 200,
  "body": "\"Total EC2 Instances (Running & Stopped): 7\""
}
```
---


### 📬 Author
Veerakumar Thangavel
DevOps Engineer – Cloud | Automation | Monitoring | Infrastructure 
