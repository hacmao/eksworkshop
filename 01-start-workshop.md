# Start workshop  

## Install kubectl  

```bash
sudo curl --silent --location -o /usr/local/bin/kubectl \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.17.11/2020-09-18/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
```

+ Kubectl completion :  

```bash
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

## Khai báo enviroment  

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text --region $AWS_REGION))
```

Ghi vào file khởi động :  

```bash
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
echo "export AZS=(${AZS[@]})" | tee -a ~/.bash_profile
```

## Lauch EKS cluster  

### Install `eksctl`  

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```

+ `eksctl` completion :  

```bash
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

## Khóa CMK  

Tạo khóa để mã hóa cho secret của kubenets:  

```bash
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
```

### File cài đặt  

Tạo file cài đặt :  

```bash
cat << EOF > eksworkshop.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl
  region: ${AWS_REGION}
  version: "1.17"

availabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t2.micro
  ssh:
    allow: true
    publicKeyName: eksworkshop

# To enable all of the control plane logs, uncomment below:
# cloudWatch:
#  clusterLogging:
#    enableTypes: ["*"]

secretsEncryption:
  keyARN: ${MASTER_ARN}
EOF
```

Chúng ta sẽ tạo một cluster có 3 node, mỗi node là một EC2 instance có instance type là `t3.small`, có ssh key là `eksworkshop`, có encryptkey là `${MASTER_ARN}` tạo từ bước trước.  
Lưu ý, nếu không setup đúng các biến môi trường như trong workshop cần tự cấu hình trong file cài đặt :)  

Tạo cluster :  

```bash
eksctl create cluster -f eksworkshop.yaml
```

Test cluster :  

```bash
kubectl get nodes 
```

[Note](https://github.com/kubernetes-sigs/aws-iam-authenticator/issues/174#issuecomment-450651720) : Một lưu ý nữa, ta cần thiết lập hai biến môi trường là `AWS_ACCESS_KEY_ID` và `AWS_SECRET_ACCESS_KEY`, để kubectl có thể xác thực, login vào cluster vừa tạo.  

Lưu worker role:  

```bash
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```
