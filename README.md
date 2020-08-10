# Delete Empty CloudWatch Log Streams

This tool can be deployed in an AWS Account in a given region and it will ensure
that all of the CloudWatch Log Groups which have a retention policy defined will
not build up large numbers of empty Log Streams (streams which previously
contained events that have since been deleted by the retention policy).

This is important to do as the number of empty Log Streams in a Log Group
increases, the ability to tail the logs in that Log Group is affected and
eventually stops working entirely.

This tool is very simple and can be deployed in a single CloudFormation
template. The template creates a Lambda function which handles the deletion of
the empty Log Streams.

# How to deploy

1. Deploy the [`DeleteEmptyCloudWatchLogStreams.yaml`](DeleteEmptyCloudWatchLogStreams.yaml)
   CloudFormation template either in the AWS web console or on the command line
   ```
   aws cloudformation deploy --stack-name DeleteEmptyCloudWatchLogStreams \
     --template-file DeleteEmptyCloudWatchLogStreams.yaml \
     --capabilities CAPABILITY_IAM
   ```
2. Invoke (run) the Lambda function one time to get it going.
   * Either in the AWS Web Console click the `Test` button in
     the AWS Lambda function (and pass it any payload, doesn't matter)
   * Or run
     ```
     function_name=$(aws cloudformation describe-stack-resources \
       --stack-name DeleteEmptyCloudWatchLogStreams \
       --logical-resource-id DeleteEmptyLogStreamsFunction \
       --query StackResources[].[PhysicalResourceId] --output text)

     aws lambda invoke --function-name ${function_name} /dev/stdout
     ```

# How does it work

The CloudFormation template deploys an AWS Lambda function. When the Lambda
function is invoked it
* Iterates over all of the AWS CloudWatch Log Groups in the account and region
* If the Log Group has a retention policy set (which tells CloudWatch logs to
  delete events older than a certain number of days) then the Lambda function
  iterates over the Log Streams within that Log Group and deletes all streams
  which are older than the what Log Group's retention policy allows
* If the Lambda function is not done in 15 minutes it has CloudWatch Event Rules
  re-invoke the function again. It continues this until it deletes all Log
  Streams older than the retention policy allows for all Log Groups that have 
  a retention policy set.
* Once the Lambda function is able to complete in 15 minutes, it schedules
  itself to run again in one day to trim new aged-out Log Streams

# How to see it running

You can use tools like [awscli v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html),
[awslogs](https://github.com/jorgebastida/awslogs) or [cw](https://www.lucagrulla.com/cw/)
to tail the logs.

You can get the CloudWatch Log Group name from CloudFormation with this command
```
log_group=$(aws cloudformation describe-stack-resources \
   --stack-name DeleteEmptyCloudWatchLogStreams \
   --logical-resource-id DeleteEmptyLogStreamsFunctionLogGroup \
   --query StackResources[].[PhysicalResourceId] --output text)
```

And use that Log Group name with commands like

```
awslogs get --start 1m --no-group --no-stream --watch ${log_group}
```

```
aws logs tail ${log_group} --since 5d
```

# Other Solutions

## aws-cloudwatch-log-minder

https://github.com/binxio/aws-cloudwatch-log-minder

This solution differs in that it
* runs hourly and fans out child Lambda invocations for each Log Group
* doesn't account for Log Groups that have enough Log Streams that it takes
  longer than the Lambda maximum execution time to complete them. The result is
  that for Log Groups with many Log Streams, the tool would run for 15 minutes
  then fail and not start up again until the next invocation in an hour.
* sets a retention policy on all Log Groups
* checks Log Streams to see if they're empty instead of just assuming that if
  they are older than the retention policy of the Log Group, that they should be
  deleted
* deploys via an S3 hosted Lambda package instead of a single CloudFormation
  template
* creates two Lambda functions
* assumes a manageable number of Log Groups instead of iterating by page in case
  there are many
* also contains a command line launchable version

## aws-cloudwatch-log-clean

https://github.com/four43/aws-cloudwatch-log-clean/blob/master/sweep_log_streams.py

This solution differs in that it
* Doesn't run in Lambda
* It doesn't use boto3 paginators

# Related Threads

https://forums.aws.amazon.com/message.jspa?messageID=862786