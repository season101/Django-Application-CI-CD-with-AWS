# Django Backend CI/CD

# Table of Content

# Resources Used

1. EC2 Instance
2. AWS CodeDeploy
3. AWS CodePipeline
4. Enterprise Github
5. IAM Roles

# CI/CD Solution Architecture

We will be proceeding with the following architecture for the CI/CD for an Django application.

![CICDDQ-CI_CD Frontend API_ Architecture Diagram .jpg](Django%20Backend%20CI%20CD/CICDDQ-CI_CD_Frontend_API__Architecture_Diagram_.jpg)

# Steps

## Repository

Setup the repository to manage credentials for automatic deployment.

### .env

This is essential to protect the credentials in Application. The easier implementation to handle this is to use .env file. Locally create the .env with right credentials as values. Then have a same copy created in deployment. Use scripts to move to appropriate places in AWS CodeDeploy.

Else, Enterprise Github Server will throw an error for bad practice:

![Untitled](Django%20Backend%20CI%20CD/Untitled.png)

The reference can be found in here:

[python-dotenv](https://pypi.org/project/python-dotenv/)

******\*\*******\*\*\*\*******\*\*******Example .env for project******\*\*******\*\*\*\*******\*\*******

```jsx
TENANT=
SECRET_KEY=
CLIENT_ID=
DEBUG=
CLIENT_SECRET=
VAULT_URL=
KEY_VAULT_SECRET=
HOST_NAME=
DB_NAME=
USER_NAME=
ALLOWED_HOSTS=
```

### .gitignore

Create .gitignore to avoid pushing unwanted files or credentials to repository.

```jsx
*.pyc
.env
```

## EC2 Instance

### Create an EC2 Instance

Things to consider:

1. Operating System: Ubuntu-jammy-22.04
2. Elastic IP address needs to be assigned

### Update Apt Repository

```bash
sudo apt update
sudo apt upgrade
```

### Install Code-Deploy Agent

[Install the CodeDeploy agent for Ubuntu Server - AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install-ubuntu.html)

```bash
sudo apt install ruby-full
sudo apt install wget
cd /home/ubuntu
wget https://bucket-name.s3.region-identifier.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
sudo service codedeploy-agent start
sudo service codedeploy-agent status
```

### Install nginx

```bash
sudo apt install nginx
```

### Install Python

```bash
sudo apt install python3-pip
sudo pip3 install virtualenv
```

### Install MSSQL ODBC V17

For the application requiring additional software do at this point. For example, we are installing MSSQL ODBC V17 driver. The reference can be found at:

[Install the Microsoft ODBC driver for SQL Server (Linux) - ODBC Driver for SQL Server](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server?view=sql-server-ver16&tabs=alpine18-install,debian17-install,debian8-install,redhat7-13-install,rhel7-offline#17)

The installation requires following bash command to be run:

```bash
if ! [[ "16.04 18.04 20.04 22.04" == *"$(lsb_release -rs)"* ]];
then
    echo "Ubuntu $(lsb_release -rs) is not currently supported.";
    exit;
fi

sudo su
curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add -

curl https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/prod.list > /etc/apt/sources.list.d/mssql-release.list

exit
sudo apt-get update
sudo ACCEPT_EULA=Y apt-get install -y msodbcsql17
# optional: for bcp and sqlcmd
sudo ACCEPT_EULA=Y apt-get install -y mssql-tools
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
source ~/.bashrc
# optional: for unixODBC development headers
sudo apt-get install -y unixodbc-dev
```

### Configure System Socket & Service File for Gunicorn

```bash
[Unit]
Description=gunicorn socket
[Socket]
ListenStream=/run/gunicorn.sock
[Install]
WantedBy=sockets.target
```

```bash
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target
[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/aws-dq-dxc-web-api
ExecStart=/home/ubuntu/aws-dq-dxc-web-api/env/bin/gunicorn \
          --access-logfile /home/ubuntu/logs/access.log \
					--error-logfile /home/ubnuntu/logs/error.log \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          DjangoAPI.wsgi:application
[Install]
WantedBy=multi-user.target
```

> Note: Setup the appropriate path and values in the above configuration.

### Configure Code Deploy Service IAM role for EC2

Create a User policy and assign the following AWS managed policy to it. Then attach it to EC2.

IAM: **AmazonEC2RoleforAWSCodeDeploy**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": ["s3:GetObject", "s3:GetObjectVersion", "s3:ListBucket"],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

### Tag EC2 Instance

It is necessary for Code deploy application deployment group to recognize the instance.

Key: Name

Value: unique name-for-ec2

## Code Deploy

It manages the deployment of artifacts into the EC2 instance. Make sure to do code-deploy agent installation first in EC2 instance. From the AWS Code Deploy Console proceed with create application.

### Create Application

Give an application name and select a compute platform to deploy the code. For this case: EC2/On-premises.

![Untitled](Django%20Backend%20CI%20CD/Untitled%201.png)

### Deployment groups

Select the currently created application. and select the Deployment groups tab. Then proceed with the Create Deployment group.

### Deployment groups: Deployment group name

![Untitled](Django%20Backend%20CI%20CD/Untitled%202.png)

Give a unique deployment group name.

### Deployment groups: Service role

![Untitled](Django%20Backend%20CI%20CD/Untitled%203.png)

Create a service role for the AWS Code Deploy. For this create a role in AWS IAM console and Attach the following AWS managed Role:

AWSCodeDeployRole

The policy should be like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:CompleteLifecycleAction",
        "autoscaling:DeleteLifecycleHook",
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeLifecycleHooks",
        "autoscaling:PutLifecycleHook",
        "autoscaling:RecordLifecycleActionHeartbeat",
        "autoscaling:CreateAutoScalingGroup",
        "autoscaling:CreateOrUpdateTags",
        "autoscaling:UpdateAutoScalingGroup",
        "autoscaling:EnableMetricsCollection",
        "autoscaling:DescribePolicies",
        "autoscaling:DescribeScheduledActions",
        "autoscaling:DescribeNotificationConfigurations",
        "autoscaling:SuspendProcesses",
        "autoscaling:ResumeProcesses",
        "autoscaling:AttachLoadBalancers",
        "autoscaling:AttachLoadBalancerTargetGroups",
        "autoscaling:PutScalingPolicy",
        "autoscaling:PutScheduledUpdateGroupAction",
        "autoscaling:PutNotificationConfiguration",
        "autoscaling:PutWarmPool",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DeleteAutoScalingGroup",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:TerminateInstances",
        "tag:GetResources",
        "sns:Publish",
        "cloudwatch:DescribeAlarms",
        "cloudwatch:PutMetricAlarm",
        "elasticloadbalancing:DescribeLoadBalancers",
        "elasticloadbalancing:DescribeInstanceHealth",
        "elasticloadbalancing:RegisterInstancesWithLoadBalancer",
        "elasticloadbalancing:DeregisterInstancesFromLoadBalancer",
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DescribeTargetHealth",
        "elasticloadbalancing:RegisterTargets",
        "elasticloadbalancing:DeregisterTargets"
      ],
      "Resource": "*"
    }
  ]
}
```

### Deployment groups: Deployment type

According to your preference select the deployment strategy. Usually this depends on the requirement of application.

![Untitled](Django%20Backend%20CI%20CD/Untitled%204.png)

### Deployment groups: Environment configuration

This depends on the solution architecture if you have multiple instance configuration for redundancy purpose. For a single instance deployment select **\*\*\*\***\*\*\*\***\*\*\*\***Amazon EC2 instance**\*\*\*\***\*\*\*\***\*\*\*\*** and select the appropriate tag you created while setting up your instance.

![Untitled](Django%20Backend%20CI%20CD/Untitled%205.png)

### Deployment groups: Agent Configuration with AWS Systems Manager

AWS Systems Manager is wonderful tool in managing SysOps work. It can be used to automate installation of code deploy agent in the EC2 instance also. However, for ease of setup, we have manually installed the code deploy agent for more control vectors. So you can proceed with the following option.

![Untitled](Django%20Backend%20CI%20CD/Untitled%206.png)

### Deployment groups: Deployment Configuration

We can opt with the one shown below. However, this can be tweaked with your own project requirements usually in the case of Auto Scaling Groups.

![Untitled](Django%20Backend%20CI%20CD/Untitled%207.png)

### Deployment groups: Load Balancer

If you have implemented the load balancer in your solution, select an appropriate target group. However, in case of single instance uncheck the Enable load balancing option.

![Untitled](Django%20Backend%20CI%20CD/Untitled%208.png)

### Deployment groups: Advanced - optional

Additional settings can be configured to enhance the deployment. Following are additional features that can be enabled with Deployment groups. You can then proceed with Create deployment group option.

![Untitled](Django%20Backend%20CI%20CD/Untitled%209.png)

## AWS CodePipeline

AWS CodePipeline helps us to create end to end CI/CD and DevOps cycle for the project. It can integrate the following phases of DevOps Cycle:

![Untitled](Django%20Backend%20CI%20CD/Untitled%2010.png)

Proceed with the Create Pipeline option from the AWS CodePipeline Console.

### Pipeline settings

Give pipeline a unique name. You can let pipeline create a service role for your project which is an easier option. For reference the following is the Service Role for the AWS CodePipeline.

```json
{
  "Statement": [
    {
      "Action": ["iam:PassRole"],
      "Resource": "*",
      "Effect": "Allow",
      "Condition": {
        "StringEqualsIfExists": {
          "iam:PassedToService": [
            "cloudformation.amazonaws.com",
            "elasticbeanstalk.amazonaws.com",
            "ec2.amazonaws.com",
            "ecs-tasks.amazonaws.com"
          ]
        }
      }
    },
    {
      "Action": [
        "codecommit:CancelUploadArchive",
        "codecommit:GetBranch",
        "codecommit:GetCommit",
        "codecommit:GetRepository",
        "codecommit:GetUploadArchiveStatus",
        "codecommit:UploadArchive"
      ],
      "Resource": "*",
      "Effect": "Allow"
    },
    {
      "Action": [
        "codedeploy:CreateDeployment",
        "codedeploy:GetApplication",
        "codedeploy:GetApplicationRevision",
        "codedeploy:GetDeployment",
        "codedeploy:GetDeploymentConfig",
        "codedeploy:RegisterApplicationRevision"
      ],
      "Resource": "*",
      "Effect": "Allow"
    },
    {
      "Action": ["codestar-connections:UseConnection"],
      "Resource": "*",
      "Effect": "Allow"
    },
    {
      "Action": [
        "elasticbeanstalk:*",
        "ec2:*",
        "elasticloadbalancing:*",
        "autoscaling:*",
        "cloudwatch:*",
        "s3:*",
        "sns:*",
        "cloudformation:*",
        "rds:*",
        "sqs:*",
        "ecs:*"
      ],
      "Resource": "*",
      "Effect": "Allow"
    },
    {
      "Action": ["lambda:InvokeFunction", "lambda:ListFunctions"],
      "Resource": "*",
      "Effect": "Allow"
    },
    {
      "Action": [
        "opsworks:CreateDeployment",
        "opsworks:DescribeApps",
        "opsworks:DescribeCommands",
        "opsworks:DescribeDeployments",
        "opsworks:DescribeInstances",
        "opsworks:DescribeStacks",
        "opsworks:UpdateApp",
        "opsworks:UpdateStack"
      ],
      "Resource": "*",
      "Effect": "Allow"
    },
    {
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:DeleteStack",
        "cloudformation:DescribeStacks",
        "cloudformation:UpdateStack",
        "cloudformation:CreateChangeSet",
        "cloudformation:DeleteChangeSet",
        "cloudformation:DescribeChangeSet",
        "cloudformation:ExecuteChangeSet",
        "cloudformation:SetStackPolicy",
        "cloudformation:ValidateTemplate"
      ],
      "Resource": "*",
      "Effect": "Allow"
    },
    {
      "Action": [
        "codebuild:BatchGetBuilds",
        "codebuild:StartBuild",
        "codebuild:BatchGetBuildBatches",
        "codebuild:StartBuildBatch"
      ],
      "Resource": "*",
      "Effect": "Allow"
    },
    {
      "Effect": "Allow",
      "Action": [
        "devicefarm:ListProjects",
        "devicefarm:ListDevicePools",
        "devicefarm:GetRun",
        "devicefarm:GetUpload",
        "devicefarm:CreateUpload",
        "devicefarm:ScheduleRun"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "servicecatalog:ListProvisioningArtifacts",
        "servicecatalog:CreateProvisioningArtifact",
        "servicecatalog:DescribeProvisioningArtifact",
        "servicecatalog:DeleteProvisioningArtifact",
        "servicecatalog:UpdateProduct"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["cloudformation:ValidateTemplate"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["ecr:DescribeImages"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "states:DescribeExecution",
        "states:DescribeStateMachine",
        "states:StartExecution"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "appconfig:StartDeployment",
        "appconfig:StopDeployment",
        "appconfig:GetDeployment"
      ],
      "Resource": "*"
    }
  ],
  "Version": "2012-10-17"
}
```

![Untitled](Django%20Backend%20CI%20CD/Untitled%2011.png)

### Advanced Settings

This is where the Artifact store can be defined and encryption method for it. You can proceed with the following options for ease of access:

![Untitled](Django%20Backend%20CI%20CD/Untitled%2012.png)

### Add Source Stage

Select the source provider. For instance, if you have your repository hosted in Enterprise GitHub Server, there is additional setting to setup connection to GitHub Enterprise Server which can be found in Miscellaneous section of this documentation. Select the connection, the repository name and the branch name and proceed with rest of the settings as follows:

![Untitled](Django%20Backend%20CI%20CD/Untitled%2013.png)

### Add build stage

For this deployment, we don’t have the build stage. Proceed with Skip build stage.

![Untitled](Django%20Backend%20CI%20CD/Untitled%2014.png)

### Add deploy stage

Proceed with the configuration for the Deploy with the appropriate configuration that you created earlier.

![Untitled](Django%20Backend%20CI%20CD/Untitled%2015.png)

### Review

Proceed to the Review stage and select option Create pipeline.

# Conclusion

Hence the CI/CD pipeline has successfully been setup. You will see the complete pipeline in AWS CodePipeline and be able to see progression through different stages as shown in the following. Further modification can be done to achieve greater setup for CI/CD with AWS.

# Miscellaneous

## appspec.yml

```bash
version: 0.0
os: linux
files:
  - source: .
    destination: /home/ubuntu/aws-dq-dxc-web-api/
file_exists_behavior: OVERWRITE
hooks:
  ApplicationStop:
    - location: scripts/stop_server.sh
      runas: root
  AfterInstall:
    - location: scripts/install_dependencies.sh
      runas: root
    - location: scripts/start_server.sh
      runas: root
```

## Scripts folder

### stop_server.sh

```bash
sudo service nginx stop
sudo service gunicorn stop
```

### start_server.sh

```bash
sudo service gunicorn restart
sudo service nginx restart
```

### install_dependencies.sh

```bash
cd /home/ubuntu/aws-dq-dxc-web-api/
rm DjangoAPI/.env
cp /home/ubuntu/.env DjangoAPI/.env
virtualenv env
source env/bin/activate
pip install --upgrade pip setuptools wheel
pip install -r requirements.txt
```

---
