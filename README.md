# check_cloudwatch_metrics

Nagios check to compare AWS CloudWatch metrics against thresholds. Also outputs performance data.

If you just need notifications, and have an existing Nagios/Icinga/Centreon setup, you can consider using this check instead of AWS CloudWatch Alarms, which each cost, as of writing, $0.10 a month once you exceed the free-tier of 10. This gives you the added benefit of integrating with your existing Nagios/Icinga notification or on-call roster.


#### Installation

You would first need to install the AWS CLI if you haven't already. If you're using RHEL/CentOS 7, the one in EPEL would do nicely:
```
yum install epel-release
yum install awscli
```
If this Nagios plugin will be run from an EC2 instance, you should, ideally, use Instance Profiles to attach a role that has CloudWatchReadOnlyAccess or similar permissions to that instance.

If this Nagios plugin will run from outside AWS, you will need an IAM user dedicated for monitoring. Though one _could_ use a standard admin user's access keys, it's extremely risky having those admin keys lying around. You would be much better off creating an IAM user with just read-only privileges, ideally on just what you need to monitor, but Amazon's "CloudWatchReadOnlyAccess" policy would do. 

To create a user, just go to IAM -> Users -> Add user, put in a User name, and just select Programmatic access (there is no need for AWS Management Console access). On the next step, click on Attach existing policies, and search for CloudWatchReadOnlyAccess and select it. Complete the steps, and note down the access/secret keys. 
Then, on Nagios/Icinga server, find out what Linux user would be running the check (eg. "nagios"), find out their home directory, and create a file called .aws/credentials with:
```
[default]
aws_secret_access_key = yourSecretKeyHere
aws_access_key_id = YOURACCESSKEY
```

(the nagios plugin also supports AWS-CLI profiles with --profile if you prefer that)

Test it out by typing this as the nagios/icinga user:
```
su -m nagios
aws cloudwatch list-metrics
```

If that worked, you can go ahead and upload this script where you usually keep your nagios plugins.


#### Constructing the parameters

This part can be tricky if you aren't familiar with CloudWatch metrics, so your best bet is to look through the Samples below to get an idea of how it works. 

You primarily just need the Namespace, Dimensions and Metric name. You can use the AWS Managment Console -> CloudWatch -> Metrics section to start with. Select the Namespace you are interested in, and then select what makes sense to you (eg. Per-Instance Metrics), and then select one of the metrics you are interested in to graph it. 

Then in the Graphed metrics options, you will have the information you need for the script arguments:

--namespace : Eg. AWS/EC2

--dimensions : Hover your mouse over the Dimensions column to get it's details, eg. InstanceId=i-01243234

--metric-name : Whatever is under MetricName

--statistic : Eg. Average

You may need to read the CloudWatch documentation to know what statistic would be useful for you if it isn't obvious. 

So you could then test the nagios plugin with something like, to warn if the average CPU utilization of this instance-id exceeeded 80 for 10 minutes:
```bash
./check_cloudwatch_metrics --region ap-southeast-2 --description "CPU usage of my test server" --namespace AWS/EC2 --metric-name CPUUtilization --dimensions InstanceId=i-0123456abcd --warning 80 --critical 95 --comparison-operator GreaterThanThreshold --statistics Average --period 600
```


#### Samples

Assuming you have a host definition with macros like:

```
define host {
		...
        host_name               your_host_name
        _CLOUDFRONT_ID          12345ABCD
        _RDS_DB                 my-database
        _ELB_NAME               my-load-balancer
		... etc
}
```

and a command definition like (the region can either be specified in your AWS-CLI profile, or as an argument here):

```
define command {
        command_name    check_cloudwatch_metrics
        command_line    $USER1$/check_cloudwatch_metrics --awsconf /var/log/nagios/.aws/config --region ap-southeast-2 $ARG1$
}
```

Here are some sample checks (note CloudFront & Route 53 checks needs to have the --region set to us-east-1 regardless of which region you normally have your resources in): 

**CloudFront error percentage**
```
	check_command           check_cloudwatch_metrics!--region us-east-1 --description "CloudFront error percentage in the past 5 minutes" --metric-name 5xxErrorRate  --namespace AWS/CloudFront --comparison-operator GreaterThanThreshold --warning 5 --critical 50 --statistics Average --period 300  --dimensions Region=Global,DistributionId=$_HOSTCLOUDFRONT_ID$
```

**RDS free disk space**
```
	check_command           check_cloudwatch_metrics!--description "RDS database free disk space in bytes" --metric-name FreeStorageSpace --namespace AWS/RDS --comparison-operator LessThanThreshold --warning 2147483648 --critical 536870912 --statistics Minimum --period 300 --dimensions DBInstanceIdentifier=$_HOSTRDS_DB$
```

**RDS CPU usage**
```
	check_command           check_cloudwatch_metrics!--description "RDS CPU usage" --metric-name CPUUtilization --namespace AWS/RDS --comparison-operator GreaterThanThreshold --warning 90 --critical 95 --statistics Average --period 300 --dimensions DBInstanceIdentifier=$_HOSTRDS_DB$
```

**RDS read latency**
```
	check_command           check_cloudwatch_metrics!--description "RDS read latency in seconds" --metric-name ReadLatency --namespace AWS/RDS --comparison-operator GreaterThanThreshold --warning 1 --critical 3 --statistics Average --period 300 --dimensions DBInstanceIdentifier=$_HOSTRDS_DB$
```

**RDS write latency**
```
	check_command           check_cloudwatch_metrics!--description "RDS write latency in seconds" --metric-name WriteLatency --namespace AWS/RDS --comparison-operator GreaterThanThreshold --warning 1 --critical 3 --statistics Average --period 300 --dimensions DBInstanceIdentifier=$_HOSTRDS_DB$
```

**ELB instances**
```
	check_command           check_cloudwatch_metrics!--description "Healthy instances" --metric-name HealthyHostCount --namespace AWS/ELB --comparison-operator LessThanThreshold --warning 1 --critical 1 --statistics Minimum --period 300 --dimensions LoadBalancerName=$_HOSTELB_NAME$
```

**ELB latency**
```
	check_command           check_cloudwatch_metrics!--description "ELB Latency in seconds" --metric-name Latency  --namespace AWS/ELB --comparison-operator GreaterThanThreshold --warning 5 --critical 8 --statistics Average --period 300 --dimensions LoadBalancerName=$_HOSTELB_NAME$
```

**ELB Backend 5XX responses**
```
	check_command           check_cloudwatch_metrics!--description "ELB Backend web server 5XX responses in the past 5 minutes" --metric-name HTTPCode_Backend_5XX  --namespace AWS/ELB --comparison-operator GreaterThanThreshold --warning 30 --critical 100 --statistics Sum --period 300 --null-value 0 --dimensions LoadBalancerName=$_HOSTELB_NAME$
```

**ELB load balancer 5XX responses**
```
	check_command           check_cloudwatch_metrics!--description "ELB load balancer 5XX responses in the past 5 minutes" --metric-name HTTPCode_ELB_5XX  --namespace AWS/ELB --comparison-operator GreaterThanThreshold --warning 20 --critical 100 --statistics Sum --period 300 --null-value 0 --dimensions LoadBalancerName=$_HOSTELB_NAME$
```

**ELB surge queue**
```
	check_command           check_cloudwatch_metrics!--description "ELB load balancer surge queue in the past 5 minutes" --metric-name SurgeQueueLength  --namespace AWS/ELB --comparison-operator GreaterThanThreshold --warning 100 --critical 1024 --statistics Maximum --period 300 --null-value 0  --dimensions LoadBalancerName=$_HOSTELB_NAME$

```
**ELB backend connection errors**
```
	check_command           check_cloudwatch_metrics!--description "ELB load balancer backend connection errors in the past 5 minutes" --metric-name BackendConnectionErrors  --namespace AWS/ELB --comparison-operator GreaterThanThreshold --warning 50 --critical 500 --statistics Sum --period 300 --null-value 0 --dimensions LoadBalancerName=$_HOSTELB_NAME$
```

**Route 53 Health check**
```
	check_command           check_cloudwatch_metrics!--region us-east-1 --description "Route 53 health check status" --metric-name HealthCheckStatus --namespace AWS/Route53 --comparison-operator LessThanThreshold --warning 1 --critical 1 --statistics Minimum --period 60 --dimensions HealthCheckId=$_HOSTROUTE53_HEALTHCHECK_ID$
```

**Route 53 Health percent**
```
	check_command           check_cloudwatch_metrics!--region us-east-1 --description "Route 53 health checkers percent" --metric-name HealthCheckPercentageHealthy --namespace AWS/Route53 --comparison-operator LessThanThreshold --warning 99 --critical 60 --statistics Minimum --period 60 --dimensions HealthCheckId=$_HOSTROUTE53_HEALTHCHECK_ID$
```

**EC2 filesystem usage among AutoScaling group with CloudWatch agent** 
```
	check_command           check_cloudwatch_metrics!--description "Highest filesystem usage percent among the web server instances" --metric-name DiskSpaceUtilization  --namespace System/Linux --comparison-operator GreaterThanThreshold --warning 85 --critical 95 --statistics Maximum --period 300 --dimensions MountPath=/,Filesystem=/dev/xvda1,AutoScalingGroupName=$_HOSTAUTOSCALING_GROUP$
```

**Lambda errors**
```
	check_command           check_cloudwatch_metrics!--description "Lambda function error count for the past 10 minutes" --metric-name Errors --namespace AWS/Lambda --comparison-operator GreaterThanOrEqualToThreshold --warning 1 --critical 1 --null-value 0 --statistics Maximum --period 600 --dimensions FunctionName=$_HOSTLAMBDA_NAME$
```


#### Script help text

```
Usage: check_cloudwatch_metrics OPTIONS...
Checks AWS CloudWatch metrics and compares them against given warning/critical thresholds

Mandatory arguments:
    --description           Any description for the nagios output text
    --metric-name           CloudWatch MetricName, eg. CPUUtilization
    --namespace             CloudWatch NameSpace, eg. AWS/RDS
    --comparison-operator   The retrieved value compared with the warning/critical thresholds, one of:
                                GreaterThanOrEqualToThreshold
                                GreaterThanThreshold
                                LessThanThreshold
                                LessThanOrEqualToThreshold
                                EqualToThreshold
                                NotEqualToThreshold
    --warning               Warning threshold number. If you just want a critical threshold, just
                            put in the same value for warning as critical
    --critical              Critical threshold number.

Optional arguments:
    --awsbin                Path to the aws binary (default uses your PATH)
    --awsconf               Path to a aws config file (default: ~/.aws/config)
    --profile               The AWS cli profile to use
    --region                The AWS region (needed if you do not have a default set in awscli)
    --statistics            One of these statistics for the average over the period:
                                Average
                                Sum
                                Minimum
                                Maximum
                            Default: Maximum
    --period                The period of time in seconds to get the statistic value for. This
                              has to be a multiple of 60
    --dimensions            CloudWatch Dimensions in a comma separated Name=Value list.
                              For example, "MountPath=/,Filesystem=/dev/xvda1"
    --unit                  Optional unit for the get-metric-statistics command
    --null-value            The value to use if there is no data for the period (=insufficient_data)
                              If you do not specify a null-value, insufficient data will return UNKNOWN
                              Specifying this is useful for say Lambda functions that run once in a while,
                              but whose errors you want to monitor. If the lambda function hasn't run during
                              the period, the Errors count would be null, but you can use --null-value 0
                              to not get UNKNOWN messages about that.

Exit status:
 0  if OK
 1  if WARNING
 2  if CRITICAL
 3  if UNKNOWN. If you are getting this, try getting the value you want
       via "aws cloudwatch get-metric-statistics" yourself, and once you get
       that working, use the same namespace & dimensions (syntax for dimensions
       differ from the aws cloudwatch command)

Example:
    check_cloudwatch_metrics --description "MySQL CPU usage" --namespace "AWS/RDS" --metric-name CPUUtilization --statistics Average --dimensions DBInstanceIdentifier=my-database --warning 80 --critical 95 --comparison-operator GreaterThanThreshold

	OK: MySQL CPU usage is 0.9175 | CPUUtilization=0.9175;80;95

```

