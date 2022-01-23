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

## EFS
- Storage Tiers
- EFS vs FSx

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

