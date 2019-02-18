---
layout: post
title: "Setting up AWS SQS as a Lambda event source from AWS CLI "
author: "Zakaria"
comments: true
---

AWS was one of the first movers into the serveless world with Lambda. Lambda functions can be a great complement to applications for integrating different components or loosening the load on backends while being cost savvy. In this post, I would like to share some AWS CLI commands that can be helpful for handling lambdas, like setting up SQS as an event source.  

# Creating the lambda

The first thing to do is to create a lambda with [`SQSEvent`](https://jar-download.com/javaDoc/com.amazonaws/aws-lambda-java-events/2.2.2/com/amazonaws/services/lambda/runtime/events/SQSEvent.html) as an input type.

{% highlight java  %}
public class DemoLambda implements RequestHandler<SQSEvent, Object> {
    public Object handleRequest(SQSEvent event, Context context) {
        event.getRecords().stream().map(SQSEvent.SQSMessage::getBody).forEach(context.getLogger()::log);
        return null;
    }
}
{% endhighlight %}

[AWS documentation](https://docs.aws.amazon.com/lambda/latest/dg/java-programming-model-handler-types.html) provides up-to date guides on how to package a java Lambda function.

After creating and packaging the lambda, the next step is to create the fuction: 

```
aws lambda  create-function --function-name demo-lambda \
--zip-file fileb://target/demo-lambda-0.1.jar \
--role arn:aws:iam::xxxxxxxxx:role/lambda-admin \
--handler com.demo.DemoLambda --runtime java8
```

where `arn:aws:iam::xxxxxxxxx:role/lambda-admin` is the arn of the lambda role (manipulating lambdas requires creating a role with Lambda privileges)

# Creating the queue and setting it as an event source

To create a SQS queue, it is enough to specify the name: `aws sqs create-queue --queue-name demo`.

 The response looks like:

{% highlight js %}
{
    "QueueUrl": "https://queue.amazonaws.com/123456789/demo"
}
{% endhighlight %}

then using the queue url, we can obtain the [arn](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html) of the queue

```
aws sqs get-queue-attributes --queue-url https://queue.amazonaws.com/123456789/demo --attribute-names QueueArn
```

{% highlight js %}
{
    "Attributes": {
        "QueueArn": "arn:aws:sqs:us-east-1:123456789:demo"
    }
}
{% endhighlight %}

the next step is to set the queue as an event source for our lambda:

`aws lambda create-event-source-mapping --function-name demo-lambda --event-source-arn arn:aws:sqs:us-east-1:123456789:demo`

# Invoking the lambda:

After hooking our queue to the Lambda function, we can invoke it by sending a message to the queue: 

`aws sqs send-message --queue-url https://sqs.us-east-1.amazonaws.com/123456789/demo --message-body 'SQS is calling lambda'`

To check that our message has reached the lambda, we need query to the lambda logs. Each lambda creates a log group whose name starts with `/aws/lambda/`. 

As intermediary steps, we need to find the log groups and streams names. 

`aws logs describe-log-groups`

{% highlight js %}

{
    "logGroups": [
        {
            "arn": "arn:aws:logs:us-east-1:123456789:log-group:/aws/lambda/demo:*", 
            "creationTime": 1547131979170, 
            "metricFilterCount": 0, 
            "logGroupName": "/aws/lambda/demo", 
            "storedBytes": 0
        }
    ]
}

{% endhighlight %}

and then using the group name we can find the stream name:

`aws logs describe-log-streams --log-group-name /aws/lambda/scp-uploader`

{% highlight js %}
{
    "logStreams": [
        {
            "firstEventTimestamp": 1549812850012, 
            "lastEventTimestamp": 1549812964252, 
            "creationTime": 1549812849361, 
            "uploadSequenceToken": "49591043180183085257539829811246426443368957060789202002", 
            "logStreamName": "2019/02/10/[$LATEST]53dab0e5596741be9e10ae534c525e62", 
            "lastIngestionTime": 1549812979302, 
            "arn": "arn:aws:logs:us-east-1:123456789:log-group:/aws/lambda/demo:log-stream:2019/02/17/[$LATEST]53dab0e5596741be9e10ae534c525e622", 
            "storedBytes": 0
        }
    ]
}
{% endhighlight %}

Using the `logStreamName`, we can check the logs to see if our message has been printed:

`aws logs get-log-events --log-group-name /aws/lambda/demo --log-stream-name '2019/02/10/[$LATEST]53dab0e5596741be9e10ae534c525e62'`


{% highlight js %}
{
    "nextForwardToken": "f/34561984021163301287653789005921321670365326296223252484", 
    "events": [
        {
            "ingestionTime": 1549812850008, 
            "timestamp": 1549812850012, 
            "message": "START RequestId: 9a2f43d0-ed94-4aff-b8aa-4b5fcae266e2 Version: $LATEST\n"
        }, 
        {
            "ingestionTime": 1549812850584, 
            "timestamp": 1549812850576, 
            "message": "SQS is calling lambda"
        }, 
}
{% endhighlight %}


The CLI commands can be useful for task automation and CI flows. All the steps above can be performed using the web UI as well.