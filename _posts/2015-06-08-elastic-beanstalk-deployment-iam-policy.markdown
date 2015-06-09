---
title: Elastic Beanstalk Deployment IAM Policy
---

This is more of a reminder to myself about a good starting place for an AWS
Elastic Beanstalk deployment IAM policy for something like TravisCI. This has
only been tested against a single-instance environment. A load balanced one
will likely require additional grants around ELBs and autoscaling groups. This assumes that you're using an S3 bucket named after your application with a suffix of `-deployments` for your deployments.

{% raw %}
You can substitute `{{appName}}` with an application name, `{{envName}}` with an environment name, and `{{accountNumber}}` with your account number.
{% endraw %}

{% highlight JSON %}
{% raw %}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketLocation",
                "s3:ListAllMyBuckets"
            ],
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::{{appName}}-deployments"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::{{appName}}-deployments/*"
        },
        {
            "Effect": "Allow",
            "Action": "elasticbeanstalk:CreateApplicationVersion",
            "Resource": "arn:aws:elasticbeanstalk:us-east-1:{{accountNumber}}:applicationversion/{{appName}}/*"
        },
        {
            "Effect": "Allow",
            "Action": "elasticbeanstalk:UpdateEnvironment",
            "Resource": "arn:aws:elasticbeanstalk:us-east-1:{{accountNumber}}:environment/{{appName}}/{{envName}}"
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudformation:CancelUpdateStack",
                "cloudformation:GetTemplate",
                "cloudformation:DescribeStackEvents",
                "cloudformation:DescribeStackResource",
                "cloudformation:DescribeStackResources",
                "cloudformation:DescribeStacks",
                "cloudformation:UpdateStack"
            ],
            "Resource": "arn:aws:cloudformation:us-east-1:{{accountNumber}}:stack/*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::elasticbeanstalk*",
                "arn:aws:s3:::elasticbeanstalk*/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeAddresses",
                "ec2:DescribeKeyPairs",
                "ec2:DescribeImages",
                "ec2:DescribeInstances",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSubnets",
                "ec2:DescribeVpcs"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "SNS:CreateTopic",
                "SNS:GetTopicAttributes",
                "SNS:ListSubscriptionsByTopic"
            ],
            "Resource": "arn:aws:sns:us-east-1:{{accountNumber}}:ElasticBeanstalkNotifications-Environment-{{envName}}"
        },
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:ResumeProcesses",
                "autoscaling:SuspendProcesses"
            ],
            "Resource": "*"
        }
    ]
}
{% endraw %}
{% endhighlight %}
