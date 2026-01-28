# Minimal Impala on AWS EC2 (End-to-End)

This guide provisions an x86_64 EC2 VM, starts a local k3d Kubernetes cluster,
installs the minimal Impala Helm chart from this repo, and runs a sample query.
It also documents how to tunnel from your laptop for `kubectl` and `impala-shell`.

## Overview: What Runs

Minimal Impala uses four services:

- `statestored`: cluster membership + state
- `catalogd`: metadata/catalog service
- `hms`: Hive Metastore (embedded Derby, quickstart config)
- `impalad` (`impalad_coord_exec`): **single daemon** acting as **coordinator + executor**

This is intentionally different from DWX-style topologies where you run separate
`impalad_coordinator` and `impalad_executor` deployments for scale and isolation.

## Prerequisites (Local)

- AWS CLI configured (`aws sts get-caller-identity`)
- SSH client
- This repo checked out at `/Users/anubhav/Desktop/impala-master`

## Step 1: Create EC2 (x86_64 with AVX)

> Impala images require x86_64 with AVX support. Apple Silicon local clusters
> will not run these images.

### 1.1 Get your public IP (to lock SSH)
```bash
curl -s https://checkip.amazonaws.com
```

### 1.2 Create key pair
```bash
KEY_NAME=impala-k8s-key-$(date +%Y%m%d%H%M%S)
aws ec2 create-key-pair --region us-east-1 --key-name "$KEY_NAME" \
  --query 'KeyMaterial' --output text > /Users/anubhav/.ssh/${KEY_NAME}.pem
chmod 600 /Users/anubhav/.ssh/${KEY_NAME}.pem
echo $KEY_NAME
```

### 1.3 Discover default VPC and subnet
```bash
VPC_ID=$(aws ec2 describe-vpcs --region us-east-1 \
  --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNET_ID=$(aws ec2 describe-subnets --region us-east-1 \
  --filters Name=vpc-id,Values=$VPC_ID Name=default-for-az,Values=true \
  --query 'Subnets[0].SubnetId' --output text)
echo $VPC_ID $SUBNET_ID
```

### 1.4 Create security group (SSH only)
```bash
MY_IP=$(curl -s https://checkip.amazonaws.com)
SG_ID=$(aws ec2 create-security-group --region us-east-1 \
  --group-name impala-k8s-sg-$(date +%Y%m%d%H%M%S) \
  --description "Impala k8s ssh" --vpc-id $VPC_ID \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --region us-east-1 \
  --group-id $SG_ID --protocol tcp --port 22 --cidr ${MY_IP}/32
echo $SG_ID
```

### 1.5 Launch Ubuntu 22.04 amd64 with k3d prereqs
```bash
AMI_ID=$(aws ssm get-parameter --region us-east-1 \
  --name /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --query 'Parameter.Value' --output text)

cat > /tmp/impala-user-data.sh <<'EOF'
#!/bin/bash
set -euxo pipefail
apt-get update
apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https git docker.io
systemctl enable --now docker
usermod -aG docker ubuntu

# kubectl
curl -sL "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
  -o /usr/local/bin/kubectl
chmod +x /usr/local/bin/kubectl

# helm
curl -sL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# k3d
curl -sL https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
EOF

INSTANCE_ID=$(aws ec2 run-instances --region us-east-1 \
  --image-id $AMI_ID --instance-type m6i.xlarge \
  --key-name $KEY_NAME --security-group-ids $SG_ID --subnet-id $SUBNET_ID \
  --associate-public-ip-address \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":50,"VolumeType":"gp2"}}]' \
  --user-data file:///tmp/impala-user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=impala-k8s-x86}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --region us-east-1 --instance-ids $INSTANCE_ID
PUBLIC_IP=$(aws ec2 describe-instances --region us-east-1 --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo $INSTANCE_ID $PUBLIC_IP
```

### 1.6 Wait for cloud-init
```bash
ssh -o StrictHostKeyChecking=no -i /Users/anubhav/.ssh/${KEY_NAME}.pem \
  ubuntu@${PUBLIC_IP} 'cloud-init status --wait'
```

## Step 2: Create k3d cluster on EC2

```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'sudo k3d cluster create impala-local --servers 1 --agents 0'
```

Set kubeconfig for the `ubuntu` user:
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'mkdir -p ~/.kube && sudo k3d kubeconfig get impala-local > ~/.kube/config && chmod 600 ~/.kube/config'
```

## Step 3: Deploy Impala (Helm)

Copy the chart to the EC2 host:
```bash
scp -i /Users/anubhav/.ssh/${KEY_NAME}.pem -r \
  /Users/anubhav/Desktop/impala-master/helm/impala \
  ubuntu@${PUBLIC_IP}:/home/ubuntu/impala-helm
```

Install the release (uses Apache prebuilt images):
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'helm upgrade --install impala /home/ubuntu/impala-helm -n impala --create-namespace \
   --set image.prefix="apache/impala:4.5.0-"'
```

Verify:
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'kubectl -n impala get pods -o wide'
```

## Step 4: Run a Sample Query (on EC2)

The easiest way is to run a short-lived `impala-shell` pod using the
quickstart client image:

```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'kubectl -n impala run impala-shell \
   --image=apache/impala:4.5.0-impala_quickstart_client \
   --restart=Never --command -- \
   impala-shell --protocol=beeswax \
   --impalad=impala-impala-impalad.impala.svc.cluster.local:21000 \
   --query "select 1"'
```

View output:
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'kubectl -n impala logs impala-shell'
```

Clean up the pod:
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'kubectl -n impala delete pod impala-shell'
```

## Step 5: Local `kubectl` from your laptop (SSH tunnel + kubeconfig)

### 5.1 Generate kubeconfig on EC2 and copy to `~/Downloads`
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'sudo k3d kubeconfig get impala-local > /tmp/impala-k3d.kubeconfig'
scp -i /Users/anubhav/.ssh/${KEY_NAME}.pem \
  ubuntu@${PUBLIC_IP}:/tmp/impala-k3d.kubeconfig \
  /Users/anubhav/Downloads/impala-k3d.kubeconfig
```

### 5.2 Create a local-use kubeconfig (127.0.0.1)
```bash
python3 - <<'PY'
from pathlib import Path
src = Path('/Users/anubhav/Downloads/impala-k3d.kubeconfig')
dst = Path('/Users/anubhav/Downloads/impala-k3d.local.kubeconfig')
text = src.read_text().replace('https://0.0.0.0:36149','https://127.0.0.1:36149')
dst.write_text(text)
print(dst)
PY
```

### 5.3 Start the tunnel
```bash
ssh -N -L 36149:localhost:36149 \
  -i /Users/anubhav/.ssh/${KEY_NAME}.pem \
  ubuntu@${PUBLIC_IP}
```

If port `36149` is busy, use `36150` and update the kubeconfig:
```bash
python3 - <<'PY'
from pathlib import Path
src = Path('/Users/anubhav/Downloads/impala-k3d.local.kubeconfig')
dst = Path('/Users/anubhav/Downloads/impala-k3d.local-36150.kubeconfig')
dst.write_text(src.read_text().replace('https://127.0.0.1:36149','https://127.0.0.1:36150'))
print(dst)
PY

ssh -N -L 36150:localhost:36149 \
  -i /Users/anubhav/.ssh/${KEY_NAME}.pem \
  ubuntu@${PUBLIC_IP}
```

### 5.4 Use kubectl locally
```bash
export KUBECONFIG=~/Downloads/impala-k3d.local.kubeconfig
kubectl get nodes
kubectl -n impala get pods
```

## Step 6: Run Queries from Local Laptop

### Option A: Local impala-shell (recommended)

Install `impala-shell` locally using pipx:
```bash
brew install pipx cyrus-sasl python@3.10
pipx install impala-shell --python /opt/homebrew/bin/python3.10
pipx ensurepath
```

Start a tunnel to the impalad service:
```bash
# Keep this open
ssh -N -L 21000:localhost:21000 \
  -i /Users/anubhav/.ssh/${KEY_NAME}.pem \
  ubuntu@${PUBLIC_IP}
```

Then run:
```bash
export PATH="$HOME/.local/bin:$PATH"
impala-shell --protocol=beeswax -i localhost:21000 -q "select 1"
```

### Option B: Run `impala-shell` inside the cluster
If you don’t want local install, run a one-off pod:
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'kubectl -n impala run impala-shell \
   --image=apache/impala:4.5.0-impala_quickstart_client \
   --restart=Never --command -- \
   impala-shell --protocol=beeswax \
   --impalad=impala-impala-impalad.impala.svc.cluster.local:21000 \
   --query "select 1"'
```

## How This Differs from DWX

Minimal setup (this guide):
- Single `impalad` (coordinator + executor)
- No separate executor/coord pools
- Scaled for simple local testing

DWX-style setup:
- Separate `impalad_coordinator` and `impalad_executor`
- Multiple executors for parallelism and scale
- More operational complexity (scheduling, resource isolation)

If you want DWX-style separation, we can add two deployments using
`apache/impala:4.5.0-impalad_coordinator` and
`apache/impala:4.5.0-impalad_executor`.

## Files Added/Modified in This Repo

### New Helm chart files
- `helm/impala/Chart.yaml`
- `helm/impala/values.yaml`
- `helm/impala/templates/_helpers.tpl`
- `helm/impala/templates/configmap.yaml`
- `helm/impala/templates/pvc.yaml`
- `helm/impala/templates/hms-deployment.yaml`
- `helm/impala/templates/hms-service.yaml`
- `helm/impala/templates/statestored-deployment.yaml`
- `helm/impala/templates/statestored-service.yaml`
- `helm/impala/templates/catalogd-deployment.yaml`
- `helm/impala/templates/catalogd-service.yaml`
- `helm/impala/templates/impalad-deployment.yaml`
- `helm/impala/templates/impalad-service.yaml`
- `helm/impala/files/hive-site.xml`

### Key modifications in the chart
- Added `statestored` RPC port `24000` service exposure.
- Added `catalogd` RPC port `26000` service exposure.
- Wired `-state_store_host` and `-catalog_service_host` to service names.
- Added `nodeSelector`, `tolerations`, `affinity` in values for x86 scheduling.

## Cleanup (Optional)

```bash
aws ec2 terminate-instances --region us-east-1 --instance-ids $INSTANCE_ID
aws ec2 delete-security-group --region us-east-1 --group-id $SG_ID
aws ec2 delete-key-pair --region us-east-1 --key-name $KEY_NAME
```
