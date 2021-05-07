
#### Assuming Roles with MFA in AWS CLI
**Scenario:**  
You have an IAM User who can assume roles in another AWS tenancy.  
How to configure AWS CLI to assume role in this another AWS tenancy using MFA?

**Steps:**
1. In your primary AWS tenancy go to `My Security Credentials` and in the `AWS IAM Credentials` section, click on `Create access key`.
2. Paste the *access key id* and the corresponding *secret access key* to the `.aws/credentials` (the example tokens and ARNs below are of course fake):
```
[default]
aws_access_key_id = TKZQYLOPHGA3ENMSKV9
aws_secret_access_key = be5rc/GRaMlqgHc6ca5bY+00c96f7j/19b09cb7e
```
3. Create a new profile (`other` in this example) in the `.aws/config`. When working using this profile, AWS CLI will assume a role in a different tenancy:
```
[default]
region = eu-west-1
output = yaml

[profile other]
region = eu-west-1
output = yaml
role_arn = arn:aws:iam::012345678910:role/devops
source_profile = default
mfa_serial = arn:aws:iam::987654321123:mfa/michal
role_session_name = michal
```

- The `role_arn` can be constructed like this: `arn:aws:iam::<another-aws-account-id>:role/<role-name>`
- The `mfa_serial` should specify the ARN of the assigned MFA device. You can find it in the `AWS IAM Credentials` section of the `My Security Credentials`.
- The `role_session_name` will be used as a user name for Audit purposes (in CloudTrail)

See more: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html#cli-configure-role-session-name

#### Assuming Roles with MFA in Terraform AWS Provider
Terraform AWS Provider Plugin does not support assuming a role with MFA ([source](https://github.com/hashicorp/terraform-provider-aws/issues/2420#issuecomment-352518083)). This can be easily workarounded by using a simple wrapper script:
```bash
#!/bin/sh
ROLE=$(aws configure get role_arn)
ROLE_SESSION_NAME=$(aws configure get role_session_name)
CRED=$(aws --output=json sts assume-role --role-arn $ROLE --role-session-name $ROLE_SESSION_NAME)

export AWS_ACCESS_KEY_ID=$(echo ${CRED} | jq -r ".Credentials.AccessKeyId")
export AWS_SECRET_ACCESS_KEY=$(echo ${CRED} | jq -r ".Credentials.SecretAccessKey")
export AWS_SESSION_TOKEN=$(echo ${CRED} | jq -r ".Credentials.SessionToken")

#echo $AWS_ACCESS_KEY_ID
#echo $AWS_SECRET_ACCESS_KEY
#echo $AWS_SESSION_TOKEN

terraform $@
```
The AWC CLI `sts assume-role` command looks into `.aws/cli/cache` directory to see if there are any associated and active temporary credentials. If not, it asks the user for the current MFA code and calls AWS API to obtain these temporary credentials. As a result, all subsequent AWS API calls done by Terraform AWS Provider Plugin will be performed on behalf of the assumed role in the other tenancy. Last but not least, the script execution has to be preceeded by setting the right profile:
```
$ export AWS_PROFILE=other
$ wrapped-terraform.sh plan
```
Because we are passing the credentials as environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`) we can keep the HCL code simple:
```hcl
provider "aws" {
  region = var.aws_region
}
```

