# EC2-Create-Notifier
This project sets up an AWS EventBridge rule to trigger a Lambda function when an EC2 instance is launched. The Lambda function sends notifications to an SNS topic.

## Prerequisites

- AWS account with necessary permissions
- AWS Management Console access

## Step 1: Create an SNS Topic

1. Sign in to the [AWS Management Console](https://aws.amazon.com/console/).
2. Navigate to **Amazon SNS**.
3. Click **Create topic**.
4. Choose **Standard** as the type.
5. Enter a **Name** (e.g., `EC2CreationAlert`).
6. Click **Create topic**.

### Create an SNS Subscription

1. Open the created SNS topic.
2. Click **Create subscription**.
3. Choose **Protocol** (e.g., **Email**).
4. Enter the **Endpoint** (your email address).
5. Click **Create subscription**.
6. Confirm the subscription via the email sent by AWS.

## Step 2: Create a Lambda Function

1. Navigate to **AWS Lambda** in the AWS Console.
2. Click **Create function**.
3. Select **Author from scratch**.
4. Enter a **Function name** (e.g., `EC2LaunchNotifier`).
5. Choose **Runtime**: `Node.js 18.x`.
6. Choose or create an **IAM role** with **SNS publish permissions**.
7. Click **Create function**.

### Add Code to Lambda

1. Open the created Lambda function.
2. In the **Code** tab, replace the content with the following:

```javascript
import { SNSClient, PublishCommand } from "@aws-sdk/client-sns";

export const handler = async (event) => {
  const { detail = {} } = event;
  const { eventName, responseElements, userIdentity } = detail;
  
  if (eventName !== "RunInstances") {
    return;
  }
  
  const instances = responseElements?.instancesSet?.items || [];
  if (instances.length === 0) {
    return;
  }

  const region = event.region || "N/A";
  const userArn = userIdentity?.arn || "N/A";
  const user = userArn.split('/').pop() || "N/A";

  const sns = new SNSClient({ region: "us-east-1" });

  const publishPromises = instances.map(async (instance) => {
    const message = `\n🚨 New EC2 Instance Launched 🚨\nInstance ID: ${instance.instanceId}\nType: ${instance.instanceType}\nLaunched by: ${user}\nRegion: ${region}\nTime: ${instance.launchTime}`;

    const params = {
      TopicArn: "arn:aws:sns:us-east-1:471112929889:EC2CreationAlert",
      Message: message,
      Subject: "EC2 Instance Created",
    };

    try {
      await sns.send(new PublishCommand(params));
      console.log(`Message sent for instance ${instance.instanceId}`);
    } catch (error) {
      console.error(`Error sending SNS message:`, error);
    }
  });

  await Promise.all(publishPromises);
};
```

3. Click **Deploy**.

## Step 3: Create an EventBridge Rule

1. Navigate to **Amazon EventBridge**.
2. Click **Rules** → **Create rule**.
3. Enter a **Name** (e.g., `EC2LaunchRule`).
4. Select **Event pattern**.
5. Under **Event source**, choose **AWS services**.
6. Select **EC2** from the **Service name** dropdown.
7. Select **AWS API Call via CloudTrail** as the **Event type**.
8. In **Specific operations**, enter `RunInstances`.
9. Click **Next**.

### Target the Lambda Function

1. Under **Target**, choose **AWS Lambda function**.
2. Select the **EC2LaunchNotifier** function.
3. Click **Next**.
4. Review settings and click **Create rule**.

## Step 4: Create an IAM Policy for EventBridge

1. Navigate to **AWS IAM**.
2. Click **Policies** → **Create policy**.
3. Select the **JSON** tab and add the following:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:EC2LaunchNotifier"
    }
  ]
}
```

4. Click **Next**, add a **Name** (e.g., `EventBridgeInvokeLambda`), and create the policy.
5. Attach this policy to the **EventBridge execution role**.

## Testing

1. Launch an EC2 instance.
2. Check if the Lambda function was triggered.
3. Verify SNS notifications in your email.
![Screenshot_20250310-115249 Gmail~2|500](https://github.com/user-attachments/assets/f07beb4d-85cc-4acd-bd55-029e1aaa715b)

---

This setup ensures automated notifications whenever an EC2 instance is launched. 🚀
