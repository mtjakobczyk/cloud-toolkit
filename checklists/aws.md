### EC2

- Instance purchasing options
  - on-demand instances
  - reserved instances
  - spot instances
    - spot instance requests (one-time or persistent)
    - spot instance interruption
    - spot blocks
    - spot fleets ( + strategies )
  - dedicated hosts
- vNIC types
  - Elastic Network Interface
  - Enhanced Networking (EN) 
    - Elastic Network Adapter 
    - Intel Virtual Function Interface
  - Elastic Fabric Adapter (EFA)
- Launch Templates (and Launch Configurations)
- Auto Scaling Groups (ASGs)
  - ASG Policies


### Elastic Load Balancing

- Load Balancer Types
  - Application Load Balancer (intelligent routing - L7)
     - Listeners, Rules, Target Groups, Health Checks
  - Network Load Balancer (fast routing - L4)
  - Classic Load Balancer (legacy, `X-Forwarded-For`)
- Sticky Sessions
- Deregistration Delay


### CloudFormation
- Template anatomy
- Input Parameters ( + aws-specific parameter types )
- Intrinsic functions
- Pseudo parameters
- Mappings ( + `Fn::FindInMap` )
- AWS::EC2::Instance - UserData property
- CloudFormation Helper Scripts
- ChangeSets


### RDS
- DB engine types
- Multi-AZ, Read replicas
- Subnet Groups
- Scaling
