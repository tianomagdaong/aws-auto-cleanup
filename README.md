# AWS Auto Cleanup

[![Codacy Badge](https://api.codacy.com/project/badge/Grade/4f20fbbb03464b9aa6c558a4415d2288)](https://app.codacy.com/app/servian/aws-auto-cleanup?utm_source=github.com&utm_medium=referral&utm_content=servian/aws-auto-cleanup&utm_campaign=Badge_Grade_Settings)

Open source application to programmatically clean your AWS resources based on a whitelist and time to live (TTL) settings.

## Table of Contents

- [Setup](#setup)
  - [Deployment](#deployment)
  - [Removal](#removal)
  - [Configuration](#configuration)
- [Tables](#tables)
- [Resource Tree](#resource-tree)

## Setup
### Deployment

To deploy this Auto Cleanup to your AWS account, follow the below steps:

1. Install Serverless
   `npm install serverless -g`
2. Install AWS CLI 
   `pip3 install awscli --upgrade --user`
3. Clone this repository 
   `git clone https://github.com/servian/aws-auto-cleanup`
4. Configure AWS CLI following the instruction at [Quickly Configuring the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html#cli-quick-configuration). Ensure the user you're configuring has the appropriate IAM permissions to create Lambda Functions, S3 Buckets, IAM Roles, and CloudFormation Stacks. It is best for administrators to deploy Auto Cleanup.
5. If you've configure the AWS CLI using a profile, open the `serverless.yml` file and modify the `provider > profile` attribute to match your profile name.
6. Change the custom `company` attribute within the `serverless.yml` file to your company name in order to prevent S3 Bucket name collision
7. Change into the Auto Cleanup directory 
   `cd aws-auto-cleanup`
8. Install Serverless plugin 
   `serverless plugin install -n serverless-python-requirements`
9. Deploy Auto Cleanup 
   `serverless deploy`
10. Invoke Auto Cleanup for the first time 
      `serverless invoke -f AutoCleanup`
11. Check Auto Cleanup logs 
      `serverless logs -f AutoCleanup`

### Removal

Auto Cleanup is deployed using the Serverless Framework which under the hood creates an AWS CloudFormation Stack. This means removal is clean and simple.

To remove Auto Cleanup from your AWS account, follow the below steps:

1. Change into the Auto Cleanup directory 
   `cd aws-auto-cleanup`
2. Remove Auto Cleanup 
   `serverless remove`

### Configuration

#### Default Values

When Auto Cleanup runs, it will populate `auto-cleanup-settings` or `auto-cleanup-whitelist` DynamoDB tables from then data files `/data/auto-cleanup-settings.json` and `/data/auto-cleanup-whitelist.json`.

#### Region

Within the `serverless.yml` file, under `provider` there is a `region` attribute. Set this attribute to your desired region.

#### Logging

Within the `serverless.yml` file, under `functions > AutoCleanup > environment` there is a `LOGLEVEL` attribute. By default, the log level is set to `INFO`. This can be changed to `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`, `CRITICAL` based on your logging requirements.

Auto Cleanup will output all resource remove logs at the `INFO` level and logs of why resources were **not** removed at the `DEBUG` level.

#### Scheduling

Within the `serverless.yml` file, under `functions > AutoCleanup > events > schedule` there is a `RATE`  and `enabled` attributes.

You can enable custom scheduling of the Lambda by following the instruction at [Schedule Expressions Using Rate or Cron](https://docs.aws.amazon.com/lambda/latest/dg/tutorial-scheduled-events-schedule-expressions.html).

The `enabled` attribute allows you to quickly enable or disable the scheduling functionality.

## Tables

Auto Cleanup uses two Amazon DynamoDB tables `auto-cleanup-settings` and `auto-cleanup-whitelist`.

### Settings

The Settings table contains all key-value pair settings used by Auto Cleanup during runtime.

The **resource** category holds all the time to live settings for each service and resource pair. By default they are all set to 7 days.

The **region** category allows users to turn region scanning on and off to either expand their search or reduce the run-time of Auto Cleanup.

By default, the below settings are automatically inserted when Auto Cleanup is run.

#### Version

The version is used to inform Auto Cleanup if new settings exist in the default data file that should be loaded into DynamoDB. If the version present in the default data file is greater than the version in DynamoDB table, the load will commence.

| Key     | Value |
| ------- | ----- |
| Version | x.x   |

#### General

| Key     | Value |
| ------- | ----- |
| Dry Run | True  |

#### Services

Table includes the `clean` attribute which informs Auto Cleanup if the service should be cleaned up or not and the `ttl` attribute which stores the time to live number of days for that service resource type pair.

| Service           | Resource Type   | Clean | TTL  |
| ----------------- | --------------- | ----- | ---- |
| CloudFormation    | Stacks          | True  | 7    |
| DynamoDB          | Tables          | True  | 7    |
| EC2               | Addresses       | True  | N/A  |
|                   | Instances       | True  | 7    |
|                   | Security Groups | True  | N/A  |
|                   | Snapshots       | True  | 7    |
|                   | Volumes         | True  | 7    |
| Elastic Beanstalk | Applications    | True  | 7    |
| EMR               | Clusters        | True  | 7    |
| IAM               | Roles           | True  | 7    |
| Lambda            | Functions       | True  | 7    |
| RDS               | Instances       | True  | 7    |
|                   | Snapshots       | True  | 7    |
| Redshift          | Clusters        | True  | 7    |
|                   | Snapshots       | True  | 7    |
| S3                | Buckets         | True  | 7    |

#### Regions

Table includes the `clean` attribute which informs Auto Cleanup if the region should be cleaned up or not.

| Region            | Clean |
| ----------------- | ----- |
| ap-northeast-1    | True  |
| ap-northeast-2    | True  |
| ap-northeast-3 \* | False |
| ap-south-1        | True  |
| ap-southeast-1    | True  |
| ap-southeast-2    | True  |
| ca-central-1      | True  |
| cn-north-1 \*     | False |
| cn-northwest-1 \* | False |
| eu-central-1      | True  |
| eu-north-1        | True  |
| eu-west-1         | True  |
| eu-west-2         | True  |
| eu-west-3         | True  |
| sa-east-1         | True  |
| us-east-1         | True  |
| us-east-2         | True  |
| us-west-1         | True  |
| us-west-2         | True  |

*Note: Some regions have `clean` set to `false` by default as they required special access from AWS*

#### Dry Run

The `dry_run` setting is used to inform Auto Cleanup if it should be removing resources it finds to have overstayed their welcome. By default, `dry_run` is set to `true`. This means that no resource removal will occur, however Auto Cleanup will output relevant logs as if it had removed resources. This allows you inspect the resources Auto Cleanup will be removing as well as giving you ample opportunity to add those that shouldn't be removed to the Whitelist table.

#### Time to Live (TTL)

In order to understand which resources have overstayed their welcome, Auto Cleanup will look at the resources created date time or last modified date time (which ever exists) and compare that to the time to live setting for that particular service resource type. If the resources was created or last modified longer than the number of days for that resources time to live setting, it will be removed.

At any time, you may modify the time to live settings for any service resource type within the `auto-cleanup-settings` Amazon DynamoDB table.

### Whitelist

The Whitelist table allows users to add their resources to prevent removal.

The Whitelist table as the following schema and comes pre-populated with Auto Cleanup resources to ensure Auto Cleanup does not remove itself:

| Column      | Format                                      | Description                                                                                                                                                   |
| ----------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| resource_id | `<service>:<resource type>:<resource name>` | Unique identifier of the resource. This is a custom format base on the service (e.g., EC2, S3), the resource type (e.g., Instance, Bucket) and resource name. |
| expire_at   | EPOCH timestamp                             | EPOCH timestamp no later than 7 days from insert date                                                                                                         |
| comment     | Text field                                  | Comment field describing the resource and why it has been whitelisted                                                                                         |
| owner_email | Email address                               | Email address of the resource owner in case they need to be contacted regarding the whitelisting                                                              |

Adding resources to the Whitelist table will ensure those resources are not removed by Auto Cleanup.

The below table lists the resource attribute that should be used for unique identification of resources for whitelisting.

| Resource                       | ID Attribute           | Example Value                                  |
| ------------------------------ | ---------------------- | ---------------------------------------------- |
| CloudFormation Stacks          | Stack Name             | `cloudformation:stack:my_cloudformation_stack` |
| DynamoDB Tables                | Table Name             | `dynamodb:table:my_dynamodb_table`             |
| EC2 Elastic IPs                | Allocation ID          | `ec2:address:eipalloc-03e6c42893296972f`       |
| EC2 Instances                  | Instance ID            | `ec2:instance:i-0326701a029dbf9d0`             |
| EC2 Security Groups            | Group ID               | `ec2:security_group:sg-09ef7b767c3ff4071`      |
| EC2 Snapshots                  | Snapshot ID            | `ec2:snapshot:snap-00c8c90db9fdceb3c`          |
| EC2 Volumes                    | Volume ID              | `ec2:volume:vol-0e1a431b9503a43aa`             |
| Elastic Beanstalk Applications | Application Name       | `elasticbeanstalk:application:my-app`          |
| EMR Clusters                   | ID                     | `emr:cluster:j-KCXVNHG2W4QK`                   |
| IAM Roles                      | Role Name              | `iam:role:auto-cleanup-role`                   |
| Lambda Functions               | Function Name          | `lambda:function:my_lambda_function`           |
| Redshift Instances             | Snapshot Identifier    | `redshift:instance:my_cluster`                 |
| Redshift Snapshots             | DB Snapshot Name       | `redshift:snapshot:my_cluster_snapshot`        |
| RDS Instances                  | DB Instance Identifier | `rds:snapshot:my_rds_instance`                 |
| RDS Snapshots                  | DB Snapshot Name       | `rds:snapshot:my_rds_instance_snapshot`        |
| S3 Buckets                     | Bucket Name            | `s3:bucket:auto-cleanup-bucket`                |

## Resource Tree

An ASCI resource tree (example below) is generated with each invocation of the application. The tree is exported into the `ResourceTreeBuckett` objects within the `serverless.yml` file.

This tree allows users to visualise their AWS resources in a simple fixed width text editor.

```bash
AWS
├── ap-southeast-2
│   ├── CloudFormation
│   │   └── Stacks
│   │       ├── auto-cleanup-dev
│   │       └── auto-cleanup-production
│   ├── DynamoDB
│   │   └── Tables
│   │       ├── auto-cleanup-settings-dev
│   │       ├── auto-cleanup-settings-production
│   │       ├── auto-cleanup-whitelist-dev
│   │       └── auto-cleanup-whitelist-production
│   ├── EC2
│   │   ├── Addresses
│   │   │   ├── eipalloc-05065c5fa7c5b481d
│   │   │   └── eipalloc-0b4c35908a9347747
│   │   ├── Instances
│   │   │   ├── i-01e008e103b99d5b2
│   │   │   └── i-07440be98bfa9a15a
│   │   ├── Security Groups
│   │   │   ├── sg-61f46719
│   │   │   └── sg-fbfa6983
│   │   ├── Snapshots
│   │   │   ├── snap-00c8c90db9fdceb3c
│   │   │   └── snap-0c416b329cacc4175
│   │   └── Volumes
│   │       ├── vol-0db31d5473b5669b0
│   │       └── vol-0e0b7da76435d07ff
│   ├── EMR
│   │   └── Clusters
│   │       ├── j-2EZVIAN6FFOT0
│   │       └── j-YR3UZD1ULRGI
│   ├── Elastic Beanstalk
│   │   └── Applications
│   │       └── my-application
│   ├── Lambda
│   │   └── Functions
│   │       ├── auto-cleanup-dev
│   │       └── auto-cleanup-production
│   └── Redshift
│       ├── Clusters
│       │   └── redshift-cluster-1
│       └── Snapshots
│           ├── rs:redshift-cluster-1-2019-04-29-19-16-08
│           └── rs:redshift-cluster-1-2019-04-30-03-16-21
├── global
│   ├── IAM
│   │   └── Roles
│   │       ├── auto-cleanup-dev-ap-southeast-2-lambdaRole
│   │       └── auto-cleanup-production-ap-southeast-2-lambdaRole
│   └── S3
│       └── Buckets
│           ├── auto-cleanup-dev-resourcetreebucket-servian
│           ├── auto-cleanup-dev-serverlessdeploymentbucket-1xa94mmahew4r
│           ├── auto-cleanup-dev-serverlessdeploymentbucket-2ftdllozuhia
│           ├── auto-cleanup-production-resourcetreebucket-servian
│           └── auto-cleanup-production-serverlessdeploymentbucke-1l0x00s8wpz9g
└── us-east-1
    ├── CloudFormation
    │   └── Stacks
    │       └── cc-iam-stack
    └── EC2
        ├── Addresses
        │   └── eipalloc-0b7a547aba879ec06
        ├── Instances
        │   └── i-0cbfb68a0d6e42c99
        └── Volumes
            └── vol-0921cdc9b3e6fc85a
```