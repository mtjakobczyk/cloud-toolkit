Dirty hands-on step by step instructions to run EKS.

It is assumed that the devops users are already created.  
See: https://github.com/mtjakobczyk/cloud-toolkit/blob/main/aws/iam/README.md 

#### Allow AWS EKS to manage other AWS services
EKS has to manage selected AWS services (EC2 instances, ...) to perform its duties.

1. Allow AWS EKS Service to manage other AWS services by creating two predefined service-linked roles:
    ```bash
    aws iam create-service-linked-role --aws-service-name eks.amazonaws.com
    aws iam create-service-linked-role --aws-service-name eks-nodegroup.amazonaws.com
    ```
    See more:
    - https://docs.aws.amazon.com/IAM/latest/UserGuide/using-service-linked-roles.html
    - https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html
    - https://docs.aws.amazon.com/eks/latest/userguide/using-service-linked-roles.html

#### Create a Role for Kubernetes admins
The role will be used to manage the lifecycle of the EKS cluster as well as act as cluster-internal admin (with `systems:masters` permissions).

1. Create a new customer-managed **IAM Policy** that allows full access to EKS and ECR
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
