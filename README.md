# lm-logs-aws-integration (beta)
This integration provides a CloudFormation template to forward logs from AWS CloudWatch to LogicMonitor.

This CloudFormation template only deploys a log forwarder (lambda function) subscribed to a specific CloudWatch logs group for LogicMonitor. 
Forwarding logs from individual AWS services, such as EC2, S3, or ELB, should be configured separately.

You will need to supply the following LogicMonitor credentials when configuring the CloudFormation stack:
* LM Access ID
* LM Access Key
* LM Account Name

### Deploying lambda using CloudFormation
[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/create/review?stackName=lm-forwarder&templateURL=https://lm-logs-forwarder.s3.amazonaws.com/latest.yaml)

### Deploying lambda using Terraform
**Sample configuration**
```tf
variable "lm_access_id" {
  description = "LogicMonitor Access Id"
}

variable "lm_access_key" {
  description = "LogicMonitor Access Key"
}

variable "lm_company_name" {
  description = "LogicMonitor Account Name"
}

# LogicMonitor Logs forwarder
resource "aws_cloudformation_stack" "lm_forwarder" {
  name         = "lm-forwarder"
  capabilities = ["CAPABILITY_IAM", "CAPABILITY_NAMED_IAM", "CAPABILITY_AUTO_EXPAND"]
  parameters   = {
    FunctionName              = "LMLogsForwarder"
    LMAccessId                = var.lm_access_id
    LMAccessKey               = var.lm_access_key
    LMCompanyName             = var.lm_company_name
    LMRegexScrub              = ""
    PermissionsBoundaryArn    = ""
  }
  template_url = "https://lm-logs-forwarder.s3.amazonaws.com/latest.yaml"
}
```
`terraform apply --var 'lm_access_id=<lm_access_id>' --var 'lm_access_key=<lm_access_key>' --var 'lm_company_name=<lm_company_name>'`

### Forwarding EC2 Instances logs
Forward EC2 logs to CloudWatch, using the [CloudWatch Logs Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html). 
**Note:** The logstream name typically defaults to the instance ID (this is expected by LogicMonitor).
After you have started recieving your EC2 logs in the desired log group:
1. Go to CloudWatch, select the desired log group of which you want to forward logs , under Actions > Create Lambda subscription filter
2. In Create Lambda subscription filter , select "Lambda Function" and choose "LMLogsForwarder" (or, whatever you named the Lambda function during stack creation) and click Start streaming.

### Forwarding S3 bucket access logs
To forward S3 bucket access logs to LogicMonitor: 
1. Under the bucket **Properties**, enable **Server access logging**.
   You will need to select a **Target bucket** where the access logs will be stored. If this target bucket doesn't exist, you need to create it. (This is different from the source bucket.)
2. Go to the target bucket, and under **Advanced settings > Events** add a notification for "All object create events". 
3. **Send to** "Lambda Function" and choose "LMLogsForwarder" (or, whatever you named the Lambda function during stack creation).

### Forwarding ELB access logs
To send ELB access logs to LogicMonitor:
1. In the EC2 navigation pane, choose Load Balancers and select your load balancer.
2. Under Attributes > Access logs, click "Configure access logs".
3. Select "Enable access logs" and specify the S3 bucket to store the logs. (You can create one, if it doesn't exist.)
4. Go to the S3 bucket (from Step 3), and under **Advanced settings > Events** add a notification for "All object create events". 
5. **Send to** "Lambda Function" and choose "LMLogsForwarder" (or, whatever you named the Lambda function during stack creation).

### Forwarding RDS logs
To send RDS logs to LogicMonitor, configure instance to send logs to cloudwatch, and create subscription filter to send logs to the LM log forwarder:
1. Follow instructions to send [standard RDS logs to cloudwatch](https://aws.amazon.com/blogs/database/monitor-amazon-rds-for-mysql-and-mariadb-logs-with-amazon-cloudwatch/) or  [enhanced RDS to cloudwatch](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_Monitoring.OS.html)
2. Go to Cloudwatch, select the desired log group of which you want to forward logs , under Actions > Create Lambda subscription filter
3. In Create Lambda subscription filter , select "Lambda Function" and choose "LMLogsForwarder" (or, whatever you named the Lambda function during stack creation) and click Start streaming.

### Forwarding Lambda logs
To send Lambda logs to LogicMonitor, go to cloudwatch and find lambda's log group, and create subscription filter to send logs to the LM log forwarder:
1. Go to Cloudwatch, select the lambda's log group of which you want to forward logs , under Actions > Create Lambda subscription filter
2. In Create Lambda subscription filter , select "Lambda Function" and choose "LMLogsForwarder" (or, whatever you named the Lambda function during stack creation) and click Start streaming.

### Send flow logs from EC2
1. Add below lines in permissions of lambda's role policy:
  "logs:CreateLogGroup",
  "logs:CreateLogStream",
  "logs:PutLogEvents"
2. Add below line in the Trust Relationship part of the role in the Service tag:
  "vpc-flow-logs.amazonaws.com"
3. A Log group in cloud watch should be created with name /aws/ec2/networkInterface
4. Use the instance id of your EC2 instance to search in Network interfaces page. Select that Network interface row and create a flow log. In create flow log Destination log group should be /aws/ec2/networkInterface and IAM role should be the role created in 1st and 2nd step.
5. In Log record format, select Custom Format. Log format should have first value as instance-id. Rest of the values can be as per your requirements. For details on different fields please refer to Available fields section on https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html. 
6. Go to /aws/ec2/networkInterface log group. In Actions > Subscription filters > Create lambda subscription filter. In lambda function select “LMLogsForwarder” (or whatever you named the Lambda function during stack creation) and provide Subscription filter name. Hit Start Streaming.

### Send flow logs from NAT Gateway
1. Add below lines in permissions of lambda's role policy:
 "logs:CreateLogGroup",
 "logs:CreateLogStream",
 "logs:PutLogEvents"
2. Add below line in the Trust Relationship part of the role in the Service tag:
 "vpc-flow-logs.amazonaws.com"
3. A Log group in cloud watch should be created with name /aws/natGateway/networkInterface
4. Use the your Nat Gateway id to search in Network interfaces page. Select that Network interface row and create a flow log. In create flow log Destination log group should be /aws/natGateway/networkInterface and IAM role should be the role created in 1st and 2nd step.
6. Go to /aws/natGateway/networkInterface log group. In Actions > Subscription filters > Create lambda subscription filter. In lambda function select “LMLogsForwarder” (or whatever you named the Lambda function during stack creation) and provide Subscription filter name. Hit Start Streaming.

### Send logs from Cloudtrail
1. On CloudTrail page click Create Trail button.
2. Provide Trail name. Unselect checkbox naming "Log file SSE-KMS encryption" if you do not want to SSE-KMS encrypt your log files.
3. Make sure to select CloudWatch Logs Enabled checkbox and provide log group name as "/aws/cloudtrail".
4. If you have existing IAM role Cloudtrail permissions, provide it as input in IAM role box. Else a new role can also be created, make sure to provide a name for the new role.
5. In the next page choose the type of logs that you would like to be collected.
6. In the next page review the provided configuration and hit Create trail button.
7. Go to Cloudwatch's Log group page and go in /aws/cloudtrail log group.
8. In Actions > Subscription filters > Create lambda subscription filter. In lambda function select “LMLogsForwarder” (or whatever you named the Lambda function during stack creation) and provide Subscription filter name. Hit Start Streaming.
9. Logs will start to propagate through lambda to LogIngest. You will be able to see logs against AWS account name resource.

### Send logs from Cloudfront
1. In Cloudfront page Select the distribution for which you would like to collect logs.
2. In Standard Logging Select "On" radio button.
3. In S3 bucket for logs Select the bucket in which you would want to store the logs.
4. Click Create Distribution.
5. Go to S3 bucket that you had selected in 3rd step.
6. Go to Properties page. Select Create event notification button in Event notifications tab.
7. Provide Event name. In Destination's Lambda function tab select “LMLogsForwarder” (or whatever you named the Lambda function during stack creation).
8. Click Save changes button.
9. You will be able to see logs at logicmonitor website against S3 bucket mentioned in 3rd step.