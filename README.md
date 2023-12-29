# AWS CloudWatch Agent for Apache Logs

This repository contains configuration files and scripts to set up the AWS CloudWatch Agent for monitoring Apache logs and sending the logs to Amazon CloudWatch.

## Table of Contents

* [AWS CloudWatch Agent for Apache Logs](#aws-cloudwatch-agent-for-apache-logs)
   * [Table of Contents](#table-of-contents)
   * [Introduction](#introduction)
   * [Getting Started 🛠️](#getting-started-️)
      * [Prerequisites 📋](#prerequisites-)
      * [Installation Steps 🚀](#installation-steps-)
         * [Create and configure an EC2 instance](#create-and-configure-an-ec2-instance)
         * [Install the CloudWatch Agent](#install-the-cloudwatch-agent)
         * [Create the CloudWatch Agent configuration file](#create-the-cloudwatch-agent-configuration-file)
         * [Configure Apache HTTP Server](#configure-apache-http-server)


## Introduction

Explain briefly what your project does and what problem it solves. Provide context and motivation for users to explore and use your AWS CloudWatch Agent setup for Apache logs.

## Getting Started 🛠️

Follow these steps to set up the CloudWatch Agent for Apache Logs:

### Prerequisites 📋

- AWS Account with CloudWatch and EC2 permissions role permission 
- Apache Web Server installed on your EC2 instances

### Installation Steps 🚀

#### Create and configure an EC2 instance
To try out CloudWatch Logs Insights, we need to have a web server that is generating logs. If you want to do this on a server you already have, skip to Step 2: Installing the CloudWatch Agent. If you want to set up a test web server so that you can try this out, follow the instructions in our Tutorial: [Install a LAMP Web Server on Amazon Linux 2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-lamp-amazon-linux-2.html).

#### Install the CloudWatch Agent
You can install the unified CloudWatch Agent via the command line with an Amazon S3 download link, AWS Systems Manager, or an AWS CloudFormation template. To install the CloudWatch Agent, follow the instructions below:

1. [Download and configure the CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/download-cloudwatch-agent-commandline.html)
2. [Create the IAM role for the CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-iam-roles-for-cloudwatch-agent-commandline.html)
3. [Attach the role to your EC2 instance](https://aws.amazon.com/blogs/security/easily-replace-or-attach-an-iam-role-to-an-existing-ec2-instance-by-using-the-ec2-console/)
#### Create the CloudWatch Agent configuration file
* After following the CloudWatch Agent installation instructions in the previous step, you can create the configuration file using the [agent configuration file wizard](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-cloudwatch-agent-configuration-file-wizard.html) with the following command:

* `sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard`
* Accept all the default choices until the wizard prompts you to select which user are you planning to run the agent. Select option **2. Root**.
* Continue accepting the default choices until it prompts you for a value for Log file path.
* Specify the following values:
  * Log file path: /var/log/httpd/error/*
  * Log group name: /aws/asg/blog/apache/error
  * Log stream name: [{instance_id}]
  * ![image](https://github.com/abaidgulshan/aws-cloudwatch-agent-apache-logs/assets/7329596/c191b62e-7585-4fd9-9f7e-d750cdd47435)
    
* When the wizard prompts for additional log files to monitor, choose 1. Yes. and specify the following values:

    * Log file path: /var/log/httpd/access/*
    * Log group name: /aws/asg/blog/apache/access
    * Log stream name: [{instance_id}]
    * ![image](https://github.com/abaidgulshan/aws-cloudwatch-agent-apache-logs/assets/7329596/1535a3cd-08ed-4789-aa73-a4d785862ffd)

#### Configure Apache HTTP Server
Run the following command to open the Apache HTTP Server configuration file:
```
sudo nano /etc/httpd/conf/httpd.conf
```
Browse the file to the log section, shown in the following screenshot.

![image](https://github.com/abaidgulshan/aws-cloudwatch-agent-apache-logs/assets/7329596/9e2a60e2-c1fd-4421-8782-97ab2721e3e9)

Your Apache HTTP Server configuration only logs messages with the “warn” flag in the default format. The default format also enables and saves access HTTP Server logs.

Modify the following line to include the /var/log/www/error/error_log destination:
```
ErrorLog "/var/log/httpd/error/access_log"
```

Below the LogLevel warn line, paste this new format for the log:
```
ErrorLogFormat "{\"time\":\"%{%usec_frac}t\", \"function\" : \"[%-m:%l]\",\"process\" : \"[pid%P]\" , \"message\" : \"%M\"}"
```

The default Apache log file format is space-separated data with one line per request. There is no native support for JSON in Apache, but with some clever formatting we can make the single line into valid JSON syntax. To do so, you can add the new LogFormat line:
```
LogFormat "{ \"time\":\"%{%Y-%m-%d}tT%{%T}t.%{msec_frac}tZ\", \"process\":\"%D\",

\"filename\":\"%f\", \"remoteIP\":\"%a\", \"host\":\"%V\", \"request\":\"%U\",

\"query\":\"%q\",\"method\":\"%m\", \"status\":\"%>s\",

\"userAgent\":\"%{User-agent}i\",\"referer\":\"%{Referer}i\"}" cloudwatch
```

Change the following access log location to logs/access/access_log:
```
CustomLog "/var/log/httpd/access/access_log" cloudwatch
```
![image](https://github.com/abaidgulshan/aws-cloudwatch-agent-apache-logs/assets/7329596/2b29c0e6-d7fd-4739-8609-b542343b3100)

Those folders don’t exist yet, and the Apache HTTP Server service daemon does not start until you create the folders. To create them, run the following commands:
```
sudo mkdir /var/log/httpd/access && /var/log/httpd/error
mkdir -p /usr/share/collectd/ && touch /usr/share/collectd/types.db
```
Restart the Apache HTTP Server service to use the new configuration:
```
sudo systemctl restart httpd
```
Tell the CloudWatch Agent to use the wizard-generated configuration file:
```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```
Now start the CloudWatch Agent:
```
sudo systemctl start amazon-cloudwatch-agent.service
```
To make sure the CloudWatch Agent starts at boot time, run the following command:
```
sudo systemctl enable amazon-cloudwatch-agent.service
```
