### Install Pre-Reqs

Amazon EKS clusters require kubectl and kubelet binaries and the aws-cli or aws-iam-authenticator
binary to allow IAM authentication for your Kubernetes cluster.


#### Install kubectl

```bash
sudo curl --silent --location -o /usr/local/bin/kubectl \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
```

#### Update awscli

Upgrade AWS CLI according to guidance in [AWS documentation](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html).

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

#### Install jq, envsubst (from GNU gettext utilities) and bash-completion

```bash
sudo yum -y install jq gettext bash-completion moreutils
```

#### Install yq for yaml processing

```bash
echo 'yq() {
  docker run --rm -i -v "${PWD}":/workdir mikefarah/yq "$@"
}' | tee -a ~/.bashrc && source ~/.bashrc
```

#### Verify the binaries are in the path and executable

```bash
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
```

#### Enable kubectl bash_completion

```bash
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

To ensure temporary credentials aren't already in place we will remove
any existing credentials file as well as disabling **AWS managed temporary credentials**:

```sh
aws cloud9 update-environment  --environment-id $C9_PID --managed-credentials-action DISABLE
rm -vf ${HOME}/.aws/credentials
```

We should configure our aws cli with our current region as default.

```sh
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text --region $AWS_REGION))

```

Check if AWS_REGION is set to desired region

```sh
test -n "$AWS_REGION" && echo AWS_REGION is "$AWS_REGION" || echo AWS_REGION is not set
```

 Let's save these into bash_profile

```sh
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
echo "export AZS=(${AZS[@]})" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```
###Attach IAM role to instance

https://www.eksworkshop.com/020_prerequisites/ec2instance/

### Validate the IAM role

Use the [GetCallerIdentity](https://docs.aws.amazon.com/cli/latest/reference/sts/get-caller-identity.html) CLI command to validate that the Cloud9 IDE is using the correct IAM role.

```bash
aws sts get-caller-identity --query Arn | grep eksworkshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
```

If the IAM role is not valid, <span style="color: red;">**DO NOT PROCEED**</span>. Go back and confirm the steps on this page.

### Expand local drive

```bash
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
    
### Create Amazon Managed Prometheus

```bash
aws amp create-workspace --alias eks-lab --region $AWS_REGION
```

Edit `CLUSTER_NAME=""` in `createIRSA-AMPIngest` and `createIRSA-AMPQuery` then run

```bash
bash ./createIRSA-AMPIngest.sh
bash ./createIRSA-AMPQuery.sh
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create ns prometheus
```

If you encounter an error deploying prometheus you may need to run the following command
### Create CRDs
```bash
kubectl create -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml
```

```bash
export SERVICE_ACCOUNT_IAM_ROLE=amp-iamproxy-ingest-role
export SERVICE_ACCOUNT_IAM_ROLE_ARN=$(aws iam get-role --role-name $SERVICE_ACCOUNT_IAM_ROLE --query 'Role.Arn' --output text)

WORKSPACE_ID=$(aws amp list-workspaces --alias eks-lab | jq .workspaces[0].workspaceId -r)

helm install prometheus-for-amp prometheus-community/prometheus -n prometheus -f ./amp_prometheus_values.yaml \
--set serviceAccounts.server.annotations."eks\.amazonaws\.com/role-arn"="${SERVICE_ACCOUNT_IAM_ROLE_ARN}" \
--set server.remoteWrite[0].url="https://aps-workspaces.${AWS_REGION}.amazonaws.com/workspaces/${WORKSPACE_ID}/api/v1/remote_write" \
--set server.remoteWrite[0].sigv4.region=${AWS_REGION}
```

### Create Amazon Managed Grafana

use AWS console
https://catalog.us-east-1.prod.workshops.aws/v2/workshops/31676d37-bbe9-4992-9cd1-ceae13c5116c/en-US/amg/setupamg-saml

### Install Loki and PromTail

```bash
helm repo add --force-update grafana https://grafana.github.io/helm-charts
    
helm upgrade --install --version 2.6.0 --namespace loki --create-namespace --values - loki grafana/loki << EOF
serviceMonitor:
  enabled: true
EOF
```

```bash
helm upgrade --install --version 3.7.0 --namespace promtail --create-namespace --values - promtail grafana/promtail << EOF
serviceMonitor:
  enabled: true
config:
  lokiAddress: http://loki-headless.loki:3100/loki/api/v1/push
EOF

```

Edit Loki to allow AMG to connect via Load Balancer
```bash
kubectl -n loki patch svc loki -p '{"spec": {"type": "LoadBalancer"}}'
```

#### Copy Demo Application Locally

```bash
cd ~/environment
git clone https://github.com/aws-containers/ecsdemo-frontend.git
git clone https://github.com/aws-containers/ecsdemo-nodejs.git
git clone https://github.com/aws-containers/ecsdemo-crystal.git
```
#### Deploy Applicaiton to EKS cluster

```bash
kubectl apply -f ./ecsdemo-frontend/kubernetes/deployment.yaml
kubectl apply -f ./ecsdemo-frontend/kubernetes/service.yaml

kubectl apply -f ./ecsdemo-nodejs/kubernetes/deployment.yaml
kubectl apply -f ./ecsdemo-nodejs/kubernetes/service.yaml

kubectl apply -f ./ecsdemo-crystal/kubernetes/deployment.yaml
kubectl apply -f ./ecsdemo-crystal/kubernetes/service.yaml
```

### Explore Grafana

#### PromQL examples

rate(node_cpu_seconds_total{mode="idle"}[5m])

rate(container_cpu_usage_seconds_total{namespace="default"}[5m])

100 - (rate(node_cpu_seconds_total{instance=~".+9100+",mode="idle"}[2m])) * 100

100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)

rate(container_cpu_usage_seconds_total{namespace=“default”[5m])

100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) 

sum by(container)(rate(container_cpu_usage_seconds_total{namespace="default"}[2m]))

% Memory available
((node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) or ((node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) / node_memory_MemTotal_bytes)) * 100

% CPU utilization
1 - avg without (mode,cpu) (rate(node_cpu_seconds_total{mode="idle"}[5m]))

####  Dashboards to import into Grafana

https://grafana.com/grafana/dashboards/1860
https://grafana.com/grafana/dashboards/10842
https://grafana.com/grafana/dashboards/10856
https://grafana.com/grafana/dashboards/10694
https://grafana.com/grafana/dashboards/7249
https://grafana.com/grafana/dashboards/3831

### cleanup


### useful links

#### Best Practices
https://aws.amazon.com/blogs/opensource/best-practices-for-migrating-self-hosted-prometheus-on-amazon-eks-to-amazon-managed-service-for-prometheus/

https://aws.github.io/aws-eks-best-practices/

#### Kubernetes Limits
https://github.com/kubernetes/community/blob/master/sig-scalability/configs-and-limits/thresholds.md

#### AMG Limits
https://docs.aws.amazon.com/grafana/latest/userguide/AMG_quotas.html

#### AMP Limits
https://docs.aws.amazon.com/prometheus/latest/userguide/AMP_quotas.html