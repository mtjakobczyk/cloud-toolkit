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
Create a new customer-managed **IAM Policy** that allows actions required by `eksctl`.

Save the following JSON as `EksAllAccess.template.json`:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        },
        {
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters"
            ],
            "Resource": [
                "arn:aws:ssm:*:<account_id>:parameter/aws/*",
                "arn:aws:ssm:*::parameter/aws/*"
            ],
            "Effect": "Allow"
        },
        {
             "Action": [
               "kms:CreateGrant",
               "kms:DescribeKey"
             ],
             "Resource": "*",
             "Effect": "Allow"
        }
    ]
}
```
Save the following JSON as `IamLimitedAccess.template.json`:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateInstanceProfile",
                "iam:DeleteInstanceProfile",
                "iam:GetInstanceProfile",
                "iam:RemoveRoleFromInstanceProfile",
                "iam:GetRole",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:AttachRolePolicy",
                "iam:PutRolePolicy",
                "iam:ListInstanceProfiles",
                "iam:AddRoleToInstanceProfile",
                "iam:ListInstanceProfilesForRole",
                "iam:PassRole",
                "iam:DetachRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:GetRolePolicy",
                "iam:GetOpenIDConnectProvider",
                "iam:CreateOpenIDConnectProvider",
                "iam:DeleteOpenIDConnectProvider",
                "iam:ListAttachedRolePolicies",
                "iam:TagRole"
            ],
            "Resource": [
                "arn:aws:iam::<account_id>:instance-profile/eksctl-*",
                "arn:aws:iam::<account_id>:role/eksctl-*",
                "arn:aws:iam::<account_id>:oidc-provider/*",
                "arn:aws:iam::<account_id>:role/aws-service-role/eks-nodegroup.amazonaws.com/AWSServiceRoleForAmazonEKSNodegroup",
                "arn:aws:iam::<account_id>:role/eksctl-managed-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:GetRole"
            ],
            "Resource": [
                "arn:aws:iam::<account_id>:role/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": [
                        "eks.amazonaws.com",
                        "eks-nodegroup.amazonaws.com",
                        "eks-fargate.amazonaws.com"
                    ]
                }
            }
        }
    ]
}
```
Render the two policy documents and execute:
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
sed "s/<account_id>/$ACCOUNT_ID/g" EksAllAccess.template.json > EksAllAccess.json
sed "s/<account_id>/$ACCOUNT_ID/g" IamLimitedAccess.template.json > IamLimitedAccess.json
aws iam create-policy --policy-name "EksAllAccess" --policy-document file://EksAllAccess.json
aws iam create-policy --policy-name "IamLimitedAccess" --policy-document file://IamLimitedAccess.json
```

Attach the IAM policies to the newly created IAM Role:
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
POLICIES=(
    'EksAllAccess',
    'IamLimitedAccess'
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
### 3. EKS (eksctl)
Prepare configuration as `ajcd2.eks.yaml` file:
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ajcd2
  region: eu-west-1

nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 2
    maxSize: 4
    volumeSize: 20
```
Create cluster and node group:
```bash
eksctl create cluster -f ajcd2.eks.yaml
```
