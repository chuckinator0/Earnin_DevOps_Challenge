# Earnin DevOps Challenge

In this challenge, we create the means with which developers can run a self-contained script that ingests raw data from S3 and writes the processed data to an RDS (Amazon Relational Database Service) instance in an existing VPC.

The script runs daily for 1 hour. It is important that this solution be cost efficient, only requiring resources when the script is running. For this reason, I have chosen to implement the script via an Amazon Lambda function. AWS Lambda allows our developers to run their job daily while we only pay for resources while they are in use. Lambda also allows us to automatically retry the script in case of failure, and dig into error logs with CloudWatch if errors persist.

I have separated steps into "Cloud Ops" and "Developer", but the goal would be to empower developers to deploy and monitor their own Lambda function in an automated way.

## Assumptions

For the purpose of this challenge, I am making the following assumptions:

* The script in question is compatible with Lambda's Python 3.6 runtime
* We have secure access to the environment variables required to run the code securely:
  * database connection credentials
  * PUBLICSUBNETID
  * PRIVATESUBNETID
  * VPCID
* The script is to run daily, and takes on the order of 1 hour to complete
* Developers have AWS CLI with proper credentials
* The lambda function will be called `dailyLambda` and the main script will be called `dailyLambda.py`

## Steps for Cloud Ops to Support Lambda Deployment

1. Create a policy and role for the lambda function using AWS CLI
2. Allow developers to deploy Lambda function (see [steps for developers](#Steps-for-Developers-to-Take-to-Integrate-with-Lambda))
3. Create a Cloudwatch Trigger for the function to execute daily

### Create Lambda Policy and Role

Create a policy that allows Lambda to interact with S3 and execute on EC2 within the same VPC as the RDS instance. Here is the policy in json format, which we can call `lambdaPolicy.json`. Note that these EC2 network interfaces allow for secure deployment of the Lambda function.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "ec2:CreateNetworkInterface",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DeleteNetworkInterface"
            ],
            "Resource": "*"
        }
    ]
}
```

To create a role called `lambdaRole` with this policy, execute this AWS CLI command:

```bash
aws iam put-role-policy --role-name lambdaRole --policy-name lambdaPolicy --policy-document file://lambdaPolicy.json
```

This role can be reused for any Lambda function that integrates with S3 and an existing VPC. Basically, it's the default AWSLambdaVPCAccessExecutionRole plus some S3 access.

### Create a Cloudwatch Trigger for the Lambda to Execute Daily

After our developers have packaged and deployed the lambda function called `dailyLambda`, we create a CloudWatch rule for the lambda function to run dialy.

```bash
aws events put-rule \
--name daily-lambda-rule \
--schedule-expression 'rate(1 day)'
```

When this rule triggers, it generates an event that looks something like:

```json
{
    "version": "0",
    "id": "53dc4d37-cffa-4f76-80c9-8b7d4a4d2eaa",
    "detail-type": "Scheduled Event",
    "source": "aws.events",
    "account": "123456789012",
    "time": "2015-10-08T16:53:06Z",
    "region": "<your region>",
    "resources": [
        "arn:aws:events:<your region>:<your account ID>:rule/daily-lambda-rule"
    ],
    "detail": {}
}
```

Add permission for the rule to apply to the lambda function:

```bash
aws lambda add-permission \
--function-name dailyLambda \
--statement-id daily-lambda-rule \
--action 'lambda:InvokeFunction' \
--principal events.amazonaws.com \
--source-arn arn:aws:events:<your region>:<your account ID>:rule/my-scheduled-rule
```

Now we need to use the `put-targets` command to add the Lambda function to this rule:

```bash
aws events put-targets --rule daily-lambda-rule --targets file://targets.json
```

where `targets.json` looks like:

```json
[
  {
    "Id": "1", 
    "Arn": "arn:aws:lambda:<your region>:<your account ID>:function:dailyLambda"
  }
]
```

* To view metrics for the rule, go to [https://console.aws.amazon.com/cloudwatch/](https://console.aws.amazon.com/cloudwatch/), Events > Rules > daily-lambda-rule
* To view logs for the funtion, go to the CloudWatch console and navigate to Logs > select log group `aws/lambda/dailyLambda` > select the log stream to view the data

## Steps for Developers to Take to Integrate with Lambda

This is a walkthrough to empower developers to deploy their script with Lambda themselves. This gives developers ownership over the deployment and monitoring of the service, but with clear guidance.

1. Package the code with all necessary libraries by using a virtual environment
2. Deploy the Lambda function using AWS CLI

### Package your code for Lambda

* Important: For consistency and reproducability, give the main function of the script the name "handler"
* Important: Use environment variables so that database credentials and other secure information isn't hardcoded

If it is important that the main function be named something else, then replace references to "handler" with the name of the main function of the script. [See this example](https://www.mydatahack.com/event-driven-data-ingestion-with-aws-lambda-s3-to-rds/) of an ETL pipeline from S3 to RDS using Lambda for more context about how to develop the script.

It is important to develop the script using the virtual environment `virtualenv` so that all necessary dependencies can be zipped together with the source code for the Lambda function to have everything it needs to run the code. [This section of Python's virtual environment docs](https://docs.python-guide.org/dev/virtualenvs/#lower-level-virtualenv)has the documentation for developing within a virtual environment.

Zip the contents of the `<virtual env folder>/lib/python3.6/site-packages/` subdirectory within your virtual environement along with our script `dailyLambda.py`:

```bash
cd <virtual env folder>/lib/python3.6/site-packages/  # navigate to installed packages
zip -r9 ../../../../dailyLambda.zip .  # zip package contents
cd ../../../../  # navigate back to main directory
zip -g dailyLambda.zip dailyLambda.py  # add our script to the zip archive
```

### Deploy Lambda Function with AWS CLI

Now we can deploy our Lambda function using the AWS CLI:

```bash
aws lambda create-function \
--region <your region> \
--function-name dailyLambda \
--zip-file fileb://dailyLambda.zip \
--role arn:aws:iam::<your account ID>:role/lambdaRole \
--environment Variables="{<database credentials>,<more database credentials>,<other environment variables>}" \   # note to use environment variables for security
--vpc-config SubnetIds=$PUBLICSUBNETID,$PRIVATESUBNETID,SecurityGroupIds=<default VPC security group ID> \
--handler dailyLambda.handler \    # remember the main function of dailyLambda.py is called handler
--runtime python3.6 \
--timeout 10 \
--memory-size 1024
```

If you need to update the funtion's configuration, use:

```bash
aws lambda update-function-configuration \
--function-name dailyLambda \
--region <new region> \
--vpc-config SubnetIds=<new info for new VPC>,<new VPC subnet info>,SecurityGroupIds=<new sg info>
```

If you need to update the funtion's code, use:

```bash
aws lambda update-function-code \
--function-name dailyLambda \
--region <your region> \
--zip-file fileb://dailyLambda.zip  # where this zip archive contains the updated code
```

To test the function manually, use:

```bash
aws lambda invoke \
--invocation-type Event \
--function-name dailyLambda \
--region <your region> \
--payload file://inputFile.txt \  # provide a test input file to transform 
outputfile.txt  # review the output file to see if the function is working as expected
```

## Discussion Questions

### What If the Job Fails?

By default, Lambda functions retry a couple of times automatically if they fail. But what if there are too many retries? To monitor this situation, we can update the function with a [Dead Letter Queue](https://docs.aws.amazon.com/lambda/latest/dg/dlq.html) configuration. To inspect the failed jobs, we could create an Amazon Simple Notifcation Service (SNS) topic and use its ARN in the Lambda function's DeadLetterConfig parameter. We would also need to modify the Lambda function's execution role to include permissions to publish to SNS.

### Things I Want to Improve

With the lambda execution role `lambdaRole`, the and the CloudWatch trigger `daily-lambda-rule` created, it would be fairly simple to automate future similar deployments of Lambda functions that operate daily. We would create a straightforward script:

* Make a function that takes the zip archive of the function as an input and would deploy the Lambda function with the proper permissions from `lambdaRole`
* Make a function that takes a CloudWatch Lambda Trigger rule `daily-lambda-rule` and the name of the lambda function and applies the trigger to the function
* Make a main function that calls the other functions to deploy the Lambda and enable the daily trigger.

We could include other important parameters like memory allocation, timeout, DeadLetterConfig (if given an ARN for an SNS resource), etc..

*HOWEVER*, managing infrastructure *procedurally* in this way goes against the emerging best practice of *declaratively* documenting infrastructure as code. For this reason, the steps of defining the execution role, CloudWatch trigger rule, and Lambda function deployment can be automated and documented as code using Terraform. [Here is the Terraform code](https://github.com/hashicorp/terraform/issues/14342#issuecomment-359185390) required to do this. I decided not to focus on this terraform code because it adds an operational complexity that would require more discussion. My goal is for developers to be able to launch and monitor their own services, and in order to do that with Terraform, we would need to have an S3 backend that is integrated with DynamoDB to lock state. This is necessary to prevent various dev teams from altering the state of the infrastructure in ways that conflict. This is not an operational complexity that is worth tackling for a simple script that runs once per day. As the needs of the developers scale, it would definitely warrant revisiting a system where developers can interact with Terraform in this way.

### Drawbacks of Lambda and Alternatives

AWS Lambda has a few general drawbacks:

* Cold start: It can take several minutes for a Lambda function to "warm up" because AWS is provisioning an EC2 and container image in the background. This doesn't matter in our use case because the data pipeline from S3 to RDS is not time sensitive to that degree.
* Computational restriction: Allocated CPU and memory are proportional to each other and limited, which is fine for our use case as a simple daily cron job. But for very large workloads, Lambda is inappropriate. As the needs of the develoopers scale, and if the nature of the job is amenable to the map-shuffle-reduce paradigm, then it may be worth looking into Amazon's Elastic Map Reduce service to run Apache Spark jobs for highly parallelized batch processing.
* Vendor lock-in. This is a bit of an operational pain, but GCP has its own version of serverless cloud functions, so it's not too much of an issue.

An alternative to using Lambda in this way is allowing developers to create their own disposable VPC in Terraform and use [VPC peering](https://www.terraform.io/docs/providers/aws/r/vpc_peering.html) to connect to the existing VPC. This allows developers to `terraform apply` their own infrastructure and determine their own computing needs while also providing the flexibility and cost-effectiveness of invoking `terraform destroy` when those resources aren't needed. It does feel a bit heavy-handed to create and destroy infrastruture daily for the script, and it also loses the observability afforded by integrating Lambda with CloudWatch and Dead Letter Queues. I prefer the Lambda approach for this reason.

Another alternative to Lambda would be to use Kubernetes. Devs could have their own cluster namespace where they deploy their job as a CronJob object. We could expose the database service to their namespace and allow them to take ownership. Kubernetes also offers by default an amazing assortment of features: efficient resource allocation, high availability, auto-recovery, zero-downtime rollouts and rollbacks, and nearly arbitrary scalability. Kubernetes would be a robust framework well beyond our simple use case. One the other hand, setting up Kubernetes would be 


### Helpful Resources

* [Example of ETL from S3 to RDS using Lambda](https://www.mydatahack.com/event-driven-data-ingestion-with-aws-lambda-s3-to-rds/)
* [Python Virtual Environments](https://docs.python-guide.org/dev/virtualenvs/#lower-level-virtualenv)
* [Using Terraform to provision Lambda for a cron job](https://www.terraform.io/docs/providers/aws/r/vpc_peering.html)
* [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)

