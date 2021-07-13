Dirty hands-on step by step instructions to run EKS.

### 1. IAM (Kubeadmins)
It is assumed that the devops users are already created.  
See: https://github.com/mtjakobczyk/cloud-toolkit/blob/main/aws/iam/README.md 

#### Create a IAM Role for Kubernetes admins
The role will be used to manage the lifecycle of the EKS cluster as well as act as cluster-internal admin (with `systems:masters` permissions).
 
##### Trust Policy for the Role
First, create a **trust policy** (aka assume role policy) which defines who can assume the role.  
In oiur case, the principals assuming the role must have the `job` tag with `kube-admin` as value.
```json
{
  "Version": "2012-10-17",
  "Statement": [
      {
          "Effect": "Allow",
          "Principal": { "AWS": "arn:aws:iam::AWS_ACCOUNT:root" },
          "Action": "sts:AssumeRole",
          "Condition": {"StringEquals": {"aws:PrincipalTag/job": "kube-admin"}}
      }
  ]
}
```
Save the policy document as `trust.template.json` and replace the placeholder with your real account ID: 
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
sed "s/AWS_ACCOUNT/$ACCOUNT_ID/g" trust.template.json > trust.json
```
##### Role
Create the role and pass the trust policy as the `--assume-role-policy-document`:
```bash
aws iam create-role --role-name "kubeadmin" --assume-role-policy-document file://trust.json
```
##### IAM Policies for the Role 
Create a new customer-managed **IAM Policy** that allows full access to EKS and ECR
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KubeadminAccess",
      "Effect": "Allow",
      "Action": [
          "eks:*",
          "ecr:*"
      ],
      "Resource": "*"
    }
  ]
}
```
Save the policy document as `kaa.json` and execute:
```bash
aws iam create-policy --policy-name "KubeadminAccess" --policy-document file://kaa.json
```

Create a new customer-managed **IAM Policy** that allows **TODO: Explain**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KubeadminPassRole",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "iam:PassRole",
        "iam:CreateServiceLinkedRole",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:CreateInstanceProfile",
        "iam:CreateOpenIDConnectProvider",
        "iam:DeleteOpenIDConnectProvider"
      ],
      "Resource": [
        "arn:aws:iam::AWS_ACCOUNT:role/*",
        "arn:aws:iam::AWS_ACCOUNT:oidc-provider/oidc.eks.eu-west-1.amazonaws.com",
        "arn:aws:iam::AWS_ACCOUNT:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/*",
        "arn:aws:ssm:*"
      ]
    }
  ]
}
```
Save the policy document as `kapr.template.json` and execute:
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
sed "s/AWS_ACCOUNT/$ACCOUNT_ID/g" kapr.template.json > kapr.json
aws iam create-policy --policy-name "KubeadminPassRole" --policy-document file://kapr.json
```

Attach the IAM policies to the newly created IAM Role:
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
POLICIES=(
    'KubeadminAccess',
    'KubeadminPassRole'
)
for policy in "${POLICIES[@]}";
do
    ARN=arn:aws:iam::$ACCOUNT_ID:policy/$policy
    aws iam attach-role-policy --role-name kubeadmin --policy-arn $ARN
done

AWS_POLICIES=(
    'AmazonEC2FullAccess' 
    'AmazonS3FullAccess' 
    'AmazonVPCFullAccess' 
    'AWSCloudFormationFullAccess'
    'AmazonSNSReadOnlyAccess'
    'IAMReadOnlyAccess' 
)
for policy in "${AWS_POLICIES[@]}";
do
    ARN=arn:aws:iam::aws:policy/$policy
    aws iam attach-role-policy --role-name kubeadmin --policy-arn $ARN
done
```
    
### Allow your devops squad to assume kubeadmin role
Users must be granted a permission to switch role to `kubeadmin` Role.

Create a **IAM policy** which defines which role can be assumed:
```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Sid": "AssumeKubeadminRole",
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::AWS_ACCOUNT:role/kubeadmin"
  }
}
```
Save the policy document as `ar.template.json` and replace the placeholder with your real account ID:
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
sed "s/AWS_ACCOUNT/$ACCOUNT_ID/g" ar.template.json > ar.json
aws iam create-policy --policy-name "AssumeKubeadminRole" --policy-document file://ar.json
```
Finally, attach the policy to the particular devops group:
```bash
GROUP_NAME=BemowoDevOpsSquad
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
aws iam attach-group-policy --group-name $GROUP_NAME --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AssumeKubeadminRole
```

### 2. IAM (EKS Service)

#### Allow AWS EKS to manage other AWS services
EKS has to manage selected AWS services (EC2 instances, ...) to perform its duties.

Allow AWS EKS Service to manage other AWS services by creating two predefined service-linked roles:
```bash
aws iam create-service-linked-role --aws-service-name eks.amazonaws.com
aws iam create-service-linked-role --aws-service-name eks-nodegroup.amazonaws.com
```
See more:
- https://docs.aws.amazon.com/IAM/latest/UserGuide/using-service-linked-roles.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html
- https://docs.aws.amazon.com/eks/latest/userguide/using-service-linked-roles.html

Create a **trust policy** (aka assume role policy) as `eks.trust-policy.json`.  
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
Create a role and attach a managed IAM policy (`AmazonEKSClusterPolicy`):
```bash
aws iam create-role --role-name "eksclusterrole" --assume-role-policy-document file://eks.trust-policy.json
aws iam attach-role-policy --role-name eksclusterrole --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```


### 3. VPC for EKS
The [CloudFormation template](./vpc.yaml) defines the VPC resources for EKS worker nodes.
```bash
aws cloudformation create-stack --region eu-west-1 --stack-name eks-vpc-stack --template-body file://vpc.yaml
aws cloudformation wait stack-create-complete --stack-name eks-vpc-stack
```
Another [CloudFormation template](./eks.yaml) defines the EKS resources and its corresponding node group.
```bash
aws cloudformation create-stack --region eu-west-1 --stack-name eks-stack --template-body file://eks.yaml
aws cloudformation wait stack-create-complete --stack-name eks-stack
```
