## EC2
- Instance Types
- Instance purchasing options
  - on-demand instances
  - reserved instances
    - convertible RIs 
    - scheduled instances
  - savings plans
  - spot instances
    - spot instance requests (one-time or persistent)
    - spot instance interruption
    - spot blocks
    - spot fleets ( + strategies )
  - dedicated hosts
- Launching an instance
  - Instance Metadata
  - User Data
- vNIC types
  - Elastic Network Interface (ENI)
  - Enhanced Networking (EN) 
    - Elastic Network Adapter 
    - Intel Virtual Function Interface
  - Elastic Fabric Adapter (EFA)
- EC2 Placement Groups (Cluster, Partition, Spread)
- Launch Templates (and Launch Configurations)
- Auto Scaling Groups (ASGs)
  - ASG Policies

## EBS
- IOPS vs Throughput
- Volume Types (SSD and HDD)
- Snapshots
- Encryption
- EC2 Hibernation
- EC2 AMIs
  - EBS vs Instance Store
- AWS Nackup

## S3
- Object Names
- Metadata
- Charges
- Consistency
- Storage Classes
- Lifecycle Management
- Object ACLs vs Bucket Policy
- Versioning
- S3 Select
- Object Locks
  - Governance and Compliance Modes
  - Retention Periods, Legal Holds
- Encryption
- Prefixes
- Multipart Uploads and Byte-Range Fetches
- Replication

## EFS
- Storage Tiers
- EFS vs FSx

## VPC
- Default VPC
- NAT Gateways
- Security Groups and NACLs
- VPC Endpoints
- VPC Peering
- PrivateLink
- AWS CloudHub
- Direct Connect
- Transit Gateway
- VPC Flow Logs

## DNS
- Registering a domain
- Record Types
  - CNAME vs alias
  - SOA
  - NS
  - CNAME
- TTL
- Health Checks for Routing Policies
- Routing Policies
  - Simple Routing
  - Multivalue Answer
  - Weighted Routing
  - Failover Routing
  - Geolocation Routing
  - Latency Routing
  - Geoproximity Routing (using Traffic Flow)

## Elastic Load Balancing
- Load Balancer Types
  - Application Load Balancer (intelligent routing - L7)
     - Listeners, Rules, Target Groups, Health Checks
  - Network Load Balancer (fast routing - L4)
  - Classic Load Balancer (legacy, `X-Forwarded-For`)
- Sticky Sessions
- Deregistration Delay

## CloudFormation
- Template anatomy
- Input Parameters ( + aws-specific parameter types )
- Intrinsic functions
- Pseudo parameters
- Mappings ( + `Fn::FindInMap` )
- AWS::EC2::Instance - UserData property
- CloudFormation Helper Scripts
- ChangeSets

## Databases
### RDS
- DB engine types
- Provisioning
  - Subnet Groups
  - DB Parameter Groups
- Pricing
- Multi-AZ
- Read replicas
- RDS Scaling
- Backups
  - Automated backups
  - Manual Snapshots
  - Restoring Backup
- Upgrades
- RDS Security
- Monitoring
  - CloudWatch
  - Enhanced Monitoring
  - Performance Insights
- Aurora
- Aurora Serverless

### DynamoDB
- DynamoDB Characteristics
  - Consistency
  - Provisioned vs On-Demand
- DAX
- Transactions
- Backups

## SQS (poll-based messaging)
- Settings
  - Visibility Timeout
 - Queue Types
   - Standard
   - FIFO
- Dead-letter queue

## SNS (push-based messaging)

## API Gateway

## Elastic Beanstalk (Application PaaS)

## Systems Manager
- Managed EC2 Instances
- Parameter Store
- Session Manager
- Automation Documents

## Monitoring
### CloudWatch
- Default Metrics vs Custom Metrics
- Default CloudWatch Period (60 seconds)
- CloudWatch Alarms
- CloudWatch Logs
- EventBridge (aka CloudWatch Events)

## Big Data
### Redshift
### Elastic Map Reduce (EMR)
### Kinesis
- Data Streams vs Data Firehose
- Data Analytics
- Kinesis vs SQS
### Athena
### Glue
- Using Glue, Athena and QuickSight to build simple Datalake
### Quicksight
### Elasticsearch

## Serverless
### Lambda
- Runtime, Permissions, Networking, Resources, Trigger
- Execution Limits

## Containers
- ECS vs EKS
### ECS
- Task Definition
- Cluster
### Fargate
- EC2 vs Fargate
- Fargate vs Lambda
