# eks-security-workshop-prep
CLI commands to run before EKS Security workshop on eksworkshop.com

### Increase disk size

```
pip3 install --user --upgrade boto3
export instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
python -c "import boto3
import os
from botocore.exceptions import ClientError 
ec2 = boto3.client('ec2')
volume_info = ec2.describe_volumes(
    Filters=[
        {
            'Name': 'attachment.instance-id',
            'Values': [
                os.getenv('instance_id')
            ]
        }
    ]
)
volume_id = volume_info['Volumes'][0]['VolumeId']
try:
    resize = ec2.modify_volume(    
            VolumeId=volume_id,    
            Size=30
    )
    print(resize)
except ClientError as e:
    if e.response['Error']['Code'] == 'InvalidParameterValue':
        print('ERROR MESSAGE: {}'.format(e))"
if [ $? -eq 0 ]; then
    sudo reboot
fi
```


### Install kubectl

```
sudo curl --silent --location -o /usr/local/bin/kubectl \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
```

### Enable kubectl bash_completion

```
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

### Update awscli

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
alias aws=/usr/local/bin/aws
```

### Install jq, envsubst (from GNU gettext utilities) and bash-completion

```
sudo yum -y install jq gettext bash-completion moreutils
```

### Install eksctl

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```

### Disabling AWS managed temporary credentials:

```
aws cloud9 update-environment  --environment-id $C9_PID --managed-credentials-action DISABLE
rm -vf ${HOME}/.aws/credentials
```

### Initialize env vars

```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text --region $AWS_REGION))
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
echo "export AZS=(${AZS[@]})" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```

### Verify IAM role

```
aws sts get-caller-identity --query Arn | grep eksworkshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
```

### Install helm

```
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

### Update kube config

```
aws eks update-kubeconfig --region us-east-2 --name eksworkshop-eksctl
```

### Create Cloud Map Namespace and services
```
OPERATION_ID=$(aws servicediscovery create-private-dns-namespace \
    --name prodcatalog-ns.svc1.cluster.local \
    --description 'App Mesh Workshop private DNS namespace' \
    --vpc vpc-xxxxxx | \
  jq -r ' .OperationId ')

NAMESPACE=$(aws servicediscovery list-namespaces | \
  jq -r ' .Namespaces[] |
    select ( .Properties.HttpProperties.HttpName == "prodcatalog-ns.svc1.cluster.local" ) | .Id ');


aws servicediscovery create-service \
  --name frontend-node \
  --description 'Discovery service for the Frontend service' \
  --namespace-id $NAMESPACE \
  --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=A,TTL=300}]' \
  --health-check-custom-config FailureThreshold=1

aws servicediscovery create-service \
  --name prodcatalog \
  --description 'Discovery service for the Prod Catalog service' \
  --namespace-id $NAMESPACE \
  --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=A,TTL=300}]' \
  --health-check-custom-config FailureThreshold=1

aws servicediscovery create-service \
  --name proddetail \
  --description 'Discovery service for the Prod Details service' \
  --namespace-id $NAMESPACE \
  --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=A,TTL=300}]' \
  --health-check-custom-config FailureThreshold=1
```
