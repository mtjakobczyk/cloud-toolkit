### VPC with 1 public subnet and 1 private subnet

```bash
RANDOM_SUFFIX=$(openssl rand -hex 4)
DEMO_PREFIX="mikes-demo-$RANDOM_SUFFIX"

aws cloudformation create-stack --stack-name "$DEMO_PREFIX-Networking" \
    --template-body file://sample-vpc.cloudformation.yaml \
    --parameters ParameterKey=VPCName,ParameterValue=lab-$RANDOM_SUFFIX-vpc  \
    --tags Key=ref:aws:environment,Value=demo
```
