---
layout: post
title:  How to create a Lambda function that sends CloudWatch alarms to a Slack channel
date:   2023-03-14
tags: aws slack lambda sns cloudwatch webhook
comments: true
---

Let's say you have a [Step Function](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html) running on AWS, and this workflow has [CloudWatch alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html) that notify you of successes, failures, and so forth. Generally speaking, CloudWatch alarms are set up with [Amazon SNS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/US_SetupSNS.html) to send email notifications. This is done by subscribing to an SNS topic and setting up the topic to email specified recipients when the CloudWatch alarm changes state (in this case, a Step Function success or failure is considered a change of state).

While I almost always recommend creating alarms for both successes and failures (if only because it helps you better monitor that your Step Function is actually running!), this also leads to inbox cluttering, especially as you increase the number of SNS topics that you are subscribed to.

<img width="600" src="https://engineer-blog.ajike.co.jp/images/posts/20190321/1.png" class="center">
<small class="center"><i>Image credit: https://ajike.github.io/lambda-to-slack/</i></small>

As an alternative, you can receive these CloudWatch notifications via Slack channel. This has a few benefits, but the most notable one in my experience has been greater visibility. It is very easy to add other collaborators to a Slack channel, and it is generally easier to track alarm history on a Slack channel versus the CloudWatch user interface or an email inbox. A Slack message also has greater customization options for a more visually appealing experience; for example, you can use a different emoji for successes versus failures, and it is easier to notice the failures while logging the successes. We will add in an "emoji mapping" when we code up the Lambda function later on in the post.

Lambda functions themselves also provide more possibilities for customization. For example, maybe you want to ping someone directly when a specific alarm goes off. This functionality can be implemented using a custom AWS Lambda function.

With all this in mind, let's get started on building out the infrastructure.

## Getting started: building the app
First, go to `api.slack.com/apps`. If you are not logged into Slack, it'll ask you to do so. If you are logged in to Slack, you should have the ability to select `Create New App` at the top right.

<img width="914" src="https://user-images.githubusercontent.com/121657180/225142038-b19a8de6-ded3-452c-9b02-a27ab91a7037.png">

You will then be prompted to name your app and assign it to a workspace in your Slack organization.

Navigate over to `Incoming Webhooks` and toggle the button to On, then click `Add New Webhook to Workspace` at the bottom.

<img width="924" src="https://user-images.githubusercontent.com/121657180/225142042-c52009c9-58b5-4973-9c78-98f87734d303.png">

Once you complete these steps, copy the webhook URL.

## Creating the Lambda function
You will want to securely store your webhook URL. To do this, go to AWS Secrets Manager -> Secrets -> `Store a new secret`.

Under `Secret type`, select `Other type of secret`. Under `Key/value pairs`, enter your secret name as the key and the webhook URL as the value. Under `Encryption key`, select `aws/secretsmanager`.

<img width="878" alt="Screen Shot 2023-03-14 at 2 43 47 PM" src="https://user-images.githubusercontent.com/121657180/225142044-cf617d7d-fae3-4fbc-af13-894e264a83cb.png">

Next, head over to AWS Lambda. This is where we will create a function that delivers CloudWatch notifications to your Slack channel.

Select `Create function`. Start by naming your function (e.g. `PostSlackSNS`) and selecting the runtime (for the purposes of this post, we will be working in Python, but you can also write your function in Node.js or Ruby). Once you've created your function, head over to Lambda -> Functions and click on your function name.

To make things easier, let's split our function out into helper functions. First, we will need a helper function that grabs the Slack webhook URL from the Secrets Manager service.

```python
import json
import boto3

# Replace this with the region you want to use in AWS
AWS_REGION = "us-west-2"

# The name of the key you created in the Secrets Manager
SLACK_SECRET_KEY = "slack_app"

def get_slack_webhook():
    # Grabs the Slack webhook URL as the value from the secret key/value pair
    secretsmanager = boto3.client("secretsmanager", region_name=AWS_REGION)
    response = secretsmanager.get_secret_value(SecretId=SLACK_SECRET_KEY)
    return json.loads(response["SecretString"])[SLACK_SECRET_KEY]
```

We will also create a helper function that uses this webhook URL to post to the Slack channel.

```python
import urllib3

def post_to_slack(msg):
    # The helper function that will post the message to the Slack channel.
    http = urllib3.PoolManager()
    url = get_slack_webhook()
    msg_encoded = json.dumps(msg).encode("utf-8")
    response = http.request("POST", url, body=msg_encoded)
```

Putting it all together (and adding in an additional helper function for user-friendly timestamp conversion), this is what our final `lambda_function.py` will look like:

```python
import datetime
import json
import urllib

import boto3
import urllib3

# Replace this with the region you want to use in AWS
AWS_REGION = "us-west-2"

# The name of the key you created in the Secrets Manager
SLACK_SECRET_KEY = "slack_app"

# Replace this with the name of your Slack channel
SLACK_CHANNEL = "slack-channel-name"

# The path to the CloudWatch alarm
PATH_TO_ALARM = f"https://{AWS_REGION}.console.aws.amazon.com/cloudwatch/home?region={AWS_REGION}#s=Alarms&alarm="

# Mapping of Slack emojis to Alarm states; change this as you see fit with your own Slack workspace emojis
ALARM_EMOJI_MAP = {"OK": ":check:", "INSUFFICIENT_DATA": ":question:", "ALARM": ":alert:"}

def get_slack_webhook():
    # Grabs the Slack webhook URL as the value from the secret key/value pair
    secretsmanager = boto3.client("secretsmanager", region_name=AWS_REGION)
    response = secretsmanager.get_secret_value(SecretId=SLACK_SECRET_KEY)
    return json.loads(response["SecretString"])[SLACK_SECRET_KEY]

def convert_timestamp(time):
    # Converts the SNS timestamp attribute to be more user-friendly
    fmt = "%Y-%m-%dT%H:%M:%S.%fZ"
    convert_time = datetime.datetime.strptime(time, fmt) + datetime.timedelta(hours=9)
    return convert_time.astimezone().strftime("%Y-%m-%d %H:%M:%S %Z")

def post_to_slack(msg):
    # The helper function that will post the message to the Slack channel.
    http = urllib3.PoolManager()
    url = get_slack_webhook()
    msg_encoded = json.dumps(msg).encode("utf-8")
    response = http.request("POST", url, body=msg_encoded)

def lambda_handler(event, context):
    msg = json.loads(event["Records"][0]["Sns"]["Message"])
    alarm_name = message["AlarmName"]
    new_state = message["NewStateValue"]
    reason = message["NewStateReason"]

    timestamp = convert_timestamp(event["Records"][0]["Sns"]["Timestamp"])

    alarm_url = PATH_TO_ALARM + urllib.parse.quote(alarm_name)

    emoji = ALARM_EMOJI_MAP[new_state]

    # Create format for slack message
    slack_message = {
        "channel": SLACK_CHANNEL,
        "username": "AWS Cloudwatch",
        "icon_emoji": emoji, # This is redundant with blocks
        "text": f"{alarm_name} has entered the {new_state} state: {reason}", # This is redundant with blocks
        "blocks": [
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"{emoji} *{alarm_name}* {emoji}\n*<{alarm_url}|Click here to view the alarm in CloudWatch.>*",
                },
            },
            {
                "type": "section",
                "fields": [
                    {"type": "mrkdwn", "text": f"*Name:*\n{alarm_name}"},
                    {"type": "mrkdwn", "text": f"*State Change:*\n{new_state}"},
                    {"type": "mrkdwn", "text": f"*Timestamp:*\n{timestamp}"},
                    {"type": "mrkdwn", "text": f"*Reason for State Change:*\n{reason}."},
                ],
            },
        ],
    }
    post_to_slack(slack_message)
```

Here, I've used Slack's block formatting functionality to compose the message. Of course, you are free to change the formatting to whatever you desire, and you can learn more about how to format Slack notifications  [here](https://api.slack.com/messaging/composing/layouts).

Before moving on, I recommend testing your function really quick just to make sure everything is working properly. Click on the `Test` tab and create a new test event. Here is an example Event JSON you can use that is similar to the actual SNS topic API output:

```json
{
  "Records": [
    {
      "EventVersion": "1.0",
      "EventSubscriptionArn": "arn:aws:sns:EXAMPLE",
      "EventSource": "aws:sns",
      "Sns": {
        "SignatureVersion": "1",
        "Timestamp": "1970-01-01T00:00:00.000Z",
        "Signature": "EXAMPLE",
        "SigningCertUrl": "EXAMPLE",
        "MessageId": "12345",
        "Message": {
          "AlarmName": "SlackAlarm",
          "NewStateValue": "OK",
          "NewStateReason": "Threshold Crossed: 1 datapoint (0.0) was not greater than or equal to the threshold (1.0)."
        },
        "MessageAttributes": {
          "Test": {
            "Type": "String",
            "Value": "TestString"
          },
          "TestBinary": {
            "Type": "Binary",
            "Value": "TestBinary"
          }
        },
        "Type": "Notification",
        "UnsubscribeUrl": "EXAMPLE",
        "TopicArn": "arn:aws:sns:EXAMPLE",
        "Subject": "TestInvoke"
      }
    }
  ]
}
```

You can now click Test and if all goes well, you should receive a message on your Slack channel.

## Adding the SNS topic as a trigger on your Lambda function
The final step in this process is to add your SNS topic to your Lambda function. 

<img width="897" src="https://user-images.githubusercontent.com/121657180/225142053-47ed8ff3-5c01-40b5-9c01-eb27045edb08.png">

Under `Function overview`, click on `Add trigger`. Under `Select a source`, search for `SNS`. Search for the SNS topic name you want to subscribe to. Once you find it, select it and click `Add`.

<img width="852" src="https://user-images.githubusercontent.com/121657180/225142047-37258305-29fc-4f8c-ad1d-ebc0d6e54b77.png">

Voila! Your SNS topic is now linked to your Slack Lambda function. From here, your Lambda function will receive messages from the SNS topic in JSON format, and from there your function will take care of the formatting and make an API POST request to your Slack webhook URL. You should now receive these messages on the Slack channel associated with that URL.

I hope this helped you. Feel free to leave a comment below if there is anything in this post that requires additional clarification. :)