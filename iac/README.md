
Creating one of the quickstart Stacks:
https://github.com/aws-quickstart/quickstart-examples/blob/main/samples/session-manager-ssh/session-manager-example.yaml
https://aws.amazon.com/blogs/infrastructure-and-automation/toward-a-bastion-less-world/

```bash
aws cloudformation create-stack --stack-name session-manager-test \
  --template-body file://session-manager-example.yaml \
  --tags Key=dance,Value=rumba \
  --parameters ParameterKey=AvailabilityZones,ParameterValue=eu-west-1a\\,eu-west-1b \
  --capabilities CAPABILITY_IAM
```
