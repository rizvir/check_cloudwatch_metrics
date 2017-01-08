# check_cloudwatch_metrics

Nagios check to compare AWS CloudWatch metrics against thresholds.

If you have an existing Nagios/Icinga/Centreon setup, you can consider using this check instead of AWS CloudWatch alarms (which each cost, as of writing, $0.10 a month once you exceed the free-tier of 10). This gives you the added benefit of integrating with your existing Nagios notification or on-call roster.

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
    --profile               The AWS cli profile to use                                                                                                        
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

