### CDK for Terraform (Python blend)
CDKTF generates JSON Terraform configuration from code.
Installation:
```bash
# requires: 
#  npm 
#  yarn (codepack enable)
#  pipenv (pip install --user pipenv)
npm install --global cdktf-cli
```
Using **Pipenv** the desired virtual environment is configured in `Pipfile`. Next to it, a `Pipfile.lock` will be automatically maintained to store the versions and hashes of libraries. In this way the used libraries will remain in the same versions, unless an update is explicitly requested (`pipenv update`). To check if there are any outdated libraries: `pipenv update --outdated`

Python-based CDKTF setup:
```bash
cdktf init --template=python --local
code .
# To set the context of the right Pipenv virtual environment:
#  - In VSCode run Command Palette and execute: Python: Select Interpreter
#  - In Terminal run: pipenv shell
```



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
