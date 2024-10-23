---
title: "How to Incorporate AWS Lambda into a Distributed System"
seoTitle: "How to Incorporate AWS Lambda into a distributed system"
seoDescription: "I'll demonstrate how to use AWS Lambda as part of a notification service, where SNS is used to publish messages to a topic and trigger Lambda function"
datePublished: Wed Oct 23 2024 07:48:20 GMT+0000 (Coordinated Universal Time)
cuid: cm2lkoiqk000209ml6jxv2nod
slug: how-to-incorporate-aws-lambda-into-a-distributed-system
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1729667799289/cf213f1b-95d8-40a5-a406-6debf34ba5d5.png
tags: tutorial, cloud, aws, python, aws-lambda

---

# Intro

Incorporating AWS Lambda into a distributed system is a powerful way to leverage the benefits of serverless computing, enabling code execution without the need to manage infrastructure. AWS Lambda excels in event-driven architectures, where functions are triggered in response to specific events, offering cost-efficiency by charging only for the compute time used. One key principle when working with Lambda is ensuring that your functions are stateless, meaning they shouldn't store state between invocations. However, triggering Lambda functions requires integration with other AWS services like SNS or SQS, adding complexity to the architecture, which makes it essential to understand the broader system design.

In this post, I'll demonstrate how to use AWS Lambda as part of a notification service, where SNS is used to publish messages to a topic, and Lambda is triggered to process those notifications. We’ll walk through configuring the necessary AWS services—IAM, SNS, SES, and Lambda—to create a simple notification service.

## AWS Lambda Overview

AWS Lambda is a serverless compute service that allows you to run code without the need to provision or manage servers. This service automatically scales based on the amount of incoming traffic, enabling your applications to handle varying workloads without requiring manual intervention. You only pay for the compute time used, making it cost-efficient and scalable for event-driven applications.

Lambda can be triggered by other AWS services like API Gateway, SNS or S3 (push model) to execute code in response to events. Depending on the event source, AWS handles whether the invocation is synchronous (where Lambda returns the response to the triggering service) or asynchronous (where Lambda processes the event and handles retries if needed).

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729667688424/905ce8b7-2756-4e5e-ae7b-57d99f5a5487.png align="center")

In a pull model, Lambda consumes data from queues or streams, periodically processing events as they come in.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729667704005/5bc2b6a5-426c-455b-9f15-eb1a5392199d.png align="center")

Security is critical in Lambda, and it uses two types of permissions: invocation permissions to allow event sources to trigger the function, and execution roles that grant Lambda access to other AWS services. These roles are managed using IAM policies.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729666872866/f42b990a-d3f6-4394-b76d-7df4f51d8df7.png align="center")

Lambda can be configured using multiple methods, including:

* AWS Management Console (UI)
    
* AWS CLI
    
* AWS SDK (language-specific API)
    
* AWS Cloudformation - infrastructure as code (JSON, YAML)
    
* AWS Serverless Application Model (templates to create different types of functions).
    

## Architecture

When a user makes a reservation, we want to send a notification to the user. In this context, AWS Lambda can be utilized as a notification service. However, we cannot directly trigger the Lambda function from our code. For example, we can use AWS Simple Notification Service (SNS) to publish a message to a topic, and then the Lambda function can be subscribed to the topic. AWS Lambda handler can encapsulate notification logic (e.g. sending an email, whitelisting the user, etc.) but it cannot send the email or sms directly. A service like Simple Email Service (SES) is needed to send the email.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729597078942/31949f38-7e6e-4a14-9b0c-20c4d23616cb.png align="center")

## Publish message to AWS SNS

In a production-ready system, our reservations service should expose API to trigger the action. However, for simplicity, we will use a simple function to simulate the reservation.

```python
import boto3
import json
import os
import random

# Initialize SNS client
sns = boto3.client(
    'sns',
    region_name=os.environ.get('REGION'),
    aws_access_key_id=os.environ.get('ACCESS_KEY_ID'),
    aws_secret_access_key=os.environ.get('SECRET_ACCESS_KEY'),
)

# Your SNS Topic ARN
sns_topic_arn = os.environ.get('TOPIC_ARN')

def publish_message(message):
    response = sns.publish(
        TopicArn=sns_topic_arn,
        Message=json.dumps({'default': json.dumps(message)}),
        MessageStructure='json',
    )
    print(f"Message published to SNS: {response['MessageId']}")


if __name__ == "__main__":
    r = random.randint(0, 1000000)
    message = {
        'event': 'Reservation',
        'user_id': '12345',
        'email': f'user+{r}@example.com',
        'body': 'Reservation made successfully',
    }
    publish_message(message)
```

We need to have installed boto3 package to use AWS SDK for Python. We can install it using pip:

```bash
pip install boto3
```

To be able to integrate with any AWS service, we need to have AWS credentials. The easiest way is to create a new IAM user with programmatic access and attach the necessary policies.

1. Log in to the AWS Management Console using your root user AWS account: [console.aws.amazon.com](http://console.aws.amazon.com).
    
2. Navigate to the IAM section.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729599772542/d57f6033-287c-4187-a900-a2f523c25fc6.png align="center")
    
3. Click on the Users tab and then click on `Create user` button.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729599721791/985ea781-92a7-4ebd-88a0-eefcbe7e81be.png align="center")
    
4. Specify the user name.
    
5. ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729599758513/947dded7-2909-42f7-bbcb-95667bd1d6d9.png align="center")
    
    Select `Attach policies directly` permission options.
    
6. Attach the following policies:
    
    * `AmazonSNSFullAccess`
        
        ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729599837629/4a06ab38-70a7-4afb-8ed9-2c2dfbc9a663.png align="center")
        
7. Continue to the next steps and create the user.
    
8. Choose the created user and click on the `Create access key` button.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729599965743/4070725a-e111-431f-bed9-e623a50c78f7.png align="center")
    
9. Download the credentials as a CSV file.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729600000096/40002d25-05e8-4ca7-ade6-af350e6e03e5.png align="center")
    
10. Set the environment variables in your system:
    

```bash
export ACCESS_KEY_ID=your_access_key_id
export SECRET_ACCESS_KEY=your_secret_access_key
```

In the next step, we need to create an AWS Simple Notification Service (SNS) topic that will be used to publish messages.

1. Navigate to the SNS service in the AWS Management Console.
    
2. Click on the `Create topic` button and click `Next`
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729600022936/f6c2ba3c-bbf6-4593-ad79-3a6ec768c12c.png align="center")
    
    .
    
3. Use the default settings and click `Create topic`.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729600041218/9d8b76fc-79f8-4f47-bb00-e43659e875cf.png align="center")
    
4. Copy the ARN of the created topic.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729600070956/df2a2dc9-5586-4b46-942b-5bd096b4f169.png align="center")
    
5. Set the environment variable in your system:
    

```bash
export TOPIC_ARN=arn:aws:sns:us-east-1:338535437683:notifications
export REGION=us-east-1
```

When we run our code:

```bash
✗ python app.py
Message published to SNS: ea014270-af41-52d3-acb4-1de1320e28fa
```

We should get a log that the message was successfully published to the SNS topic.

In the next section, we will add the AWS Lambda function that consumes events from the topic.

## AWS SES config

Before we prepare AWS lambda function, we need to configure the AWS Simple Email Service (SES) to send emails. The default SES service is in the sandbox mode, which means that you can only send emails to verified email addresses. To send emails to any email address, you need to request production access. For our example, we will use the sandbox mode.

1. Navigate to the SES service in the AWS Management Console.
    
2. Pass the email address that you want to recieve messages in the sandbox mode.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729604794217/5cc317b0-38f8-4cfc-a61b-530103810884.png align="center")
    
3. Pass the domain name (however, it is not necessary for the sandbox mode).
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729604809738/b301a466-74f5-4a60-8abe-48d38fe01555.png align="center")
    
4. Finish the verification process
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729604826481/748d4508-8158-436c-bef6-9f1f046c42cd.png align="center")
    
5. Confirm the email address in you mailbox.
    
6. Send test email.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729604871161/288e263d-cd20-47d4-ad86-e21132b0397c.png align="center")
    

To move out of the SES sandbox and send emails to any recipient, you need to request production access and provide verified domain you own.

## AWS Lambda Handler

Now, we can create the Lambda function that will be triggered by the SNS topic. However, we need to have IAM role that will allow the Lambda function to be triggered by the SNS topic.

1. Navigate to the IAM service in the AWS Management Console.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729605787392/32c99eca-adc4-43bd-82dc-1a0ead4cd02b.png align="center")
    
2. Click on the `Roles` tab and then click on the `Create role` button.
    
3. Choose the `Lambda` service as the service or use case.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729605804558/7367a40d-01dc-450d-b3b2-e4e5dac2a80a.png align="center")
    
4. Attach the following policies:
    
    * `AWSLambdaBasicExecutionRole`
        
    * `AmazonSESFullAccess`
        
        ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729605856815/80c438d1-229d-4fff-9550-0557a727a329.png align="center")
        
5. Specify the name (`LambdaUseSES` in our case) of the role and click on the `Create role` button.
    

Having the role created, we can now create the Lambda function.

1. Navigate to the Lambda service in the AWS Management Console.
    
2. Click on the `Create function` button.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729605919715/0961d1c2-77bb-4a03-b0be-fc8bdcf18424.png align="center")
    
3. Choose the `Use a blueprint` option and search for the `sns` blueprint. We use `py3.10` SNS integration.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729605935797/80529dc5-b1d1-4bbc-a9e3-90ca006f3e09.png align="center")
    
4. Specify the name of the function (`notification-handler` in our case).
    
5. Choose the `Use an existing role` option and select the role `LambdaUseSES` we have created for our lambda.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729605963519/016d091a-e4ec-461d-9e2d-6c85777d947c.png align="center")
    
6. In the SNS trigger section, select the SNS topic `notifications` we created. It will automatically create the subscription.
    
7. ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729605982228/25e31d05-0531-418a-9d6c-ca882b016da6.png align="center")
    
    The function code we will update later.
    
8. Click on the `Create function` button.
    

Our AWS lambda function is now configured to be triggered by SNS messages from `notifications` topic.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729606103091/a17162fc-d36e-4fee-907a-53889d7bdf73.png align="center")

The last step is to update the Lambda function code. We will use the following code:

```python
import json
import boto3

client = boto3.client('ses', region_name='eu-central-1')

def lambda_handler(event, context):
    data = json.loads(event['Records'][0]['Sns']['Message'])
    body = data.get('body')
    email = data.get('email')
    
    response = client.send_email(
    Destination={
        'ToAddresses': ['verified@mail.com']
    },
    Message={
        'Body': {
            'Text': {
                'Charset': 'UTF-8',
                'Data': body,
            }
        },
        'Subject': {
            'Charset': 'UTF-8',
            'Data': 'Reservation email to:' + email,
        },
    },
    Source='verified@mail.com'
    )
    print("Send mail to:" + email)
    return {
        'statusCode': 200,
        'body': json.dumps("Email Sent Successfully. MessageId is: " + response['MessageId'])
    }
```

Inside Lambda function, we can use the `boto3` package to send emails using the SES service without any additional configuration. Within the code we need to log the email address that we want to send the email to. As we are in the sandbox mode, we can only send emails to verified email addresses. So, the outgoing email address must be the one we verified in the SES service. When we are ready with the code, we can update the Lambda function code by clicking `Deploy`. The new version of the Lambda function is ready to be triggered by the SNS topic.

To test it end to end we can execute our `app.py` script:

```bash
$ python app.py
Message published to SNS: 87633e01-380e-5060-a9a4-619089a9bc6a
```

The email should have arrived to our mailbox. However, providing Lambda function with `AWSLambdaBasicExecutionRole` role enables CloudWatch logs access. How to check it?

1. Navigate to your Lambda function page.
    
2. Click on the `Monitor` tab.
    
3. Click on the `View logs in CloudWatch` button.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729606643977/080c382b-e01a-4a8a-863d-9f4df4641e99.png align="center")
    
4. Click on the latest log stream (each deployed version of your lambda would have a separate log stream).
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729606673221/fcf1584e-2a77-4005-bd0a-75b787958ef2.png align="center")
    
5. Check the logs.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729606679939/beff1094-2a6d-42f3-bf55-019db9d2c434.png align="center")

We can see the request `21c1841e-a6c2-4ecc-9856-fe6a1682454d` and successful `Send mail to:`[`user+577461@example.com`](mailto:user+577461@example.com) log. The email message can also be found in my mailbox.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729606785629/cd905a0b-507c-47a7-9780-5539252cba81.png align="center")

## Conclusion

In conclusion, while our example Lambda function is straightforward, a production-ready version should account for additional complexities, such as retry mechanisms, decision-making around notification types (email, SMS, push), and robust error handling. It's essential to think through these scenarios to ensure reliability and scalability. If you're concerned about costs, tools like the [AWS Pricing Calculator](https://calculator.aws) can help estimate Lambda expenses based on your usage.

While AWS Lambda is a powerful tool for serverless event-driven systems, it's important to recognize that it's not a one-size-fits-all solution. Understanding its limitations and the specific use cases where it excels is key to making the most of this service in your distributed system architecture.