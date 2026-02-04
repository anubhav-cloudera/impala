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

## Step 4b: Run 1G TPC-DS Load and Queries

This runs the 1 GB (SF=1) TPC-DS data generator, loads Parquet tables, and
executes a smoke query. It uses the quickstart client image to avoid local
dependencies.

### 4b.1 Create the loader pod (generates + loads)
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} <<'EOF'
kubectl -n impala delete pod tpcds-loader --ignore-not-found

cat <<'YAML' | kubectl -n impala apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tpcds-loader
spec:
  restartPolicy: Never
  containers:
    - name: loader
      image: apache/impala:4.5.0-impala_quickstart_client
      command: ["/bin/bash","-lc"]
      args:
        - |
          set -euo pipefail
          IMPALA_HOST=impala-impala-impalad.impala.svc.cluster.local
          IMPALA_PORT=21000
          IMPALA_SHELL="impala-shell --protocol=beeswax -i ${IMPALA_HOST}:${IMPALA_PORT}"

          IMPALA_TOOLCHAIN_BASE=https://native-toolchain.s3.amazonaws.com/build/7-f2ddef91e9/
          TPCDS_VERSION=2.1.0
          TPCDS_TARBALL=tpc-ds-${TPCDS_VERSION}-gcc-4.9.2-ec2-package-ubuntu-18-04.tar.gz
          TPCDS_URL=${IMPALA_TOOLCHAIN_BASE}tpc-ds/${TPCDS_VERSION}-gcc-4.9.2/${TPCDS_TARBALL}

          WAREHOUSE_EXTERNAL_DIR=/user/hive/warehouse/external
          TPCDS_RAW_DIR=${WAREHOUSE_EXTERNAL_DIR}/tpcds_raw
          mkdir -p ${TPCDS_RAW_DIR}

          if [ ! -f ${TPCDS_RAW_DIR}/generated ]; then
            curl -fL ${TPCDS_URL} --output /tmp/tpcds.tar.gz
            tar xzf /tmp/tpcds.tar.gz -C /tmp
            cd /tmp/tpc-ds-${TPCDS_VERSION}/bin
            ./dsdgen -force -verbose -scale 1
            for FILE in *.dat; do
              FILE_DIR=${TPCDS_RAW_DIR}/${FILE%.dat}
              rm -rf "${FILE_DIR}"
              mkdir -p "${FILE_DIR}"
              mv "${FILE}" "${FILE_DIR}"
            done
            touch ${TPCDS_RAW_DIR}/generated
          fi

          for i in $(seq 120); do
            if ${IMPALA_SHELL} -q "select version()"; then
              break
            fi
            sleep 2
          done

          ${IMPALA_SHELL} -f /opt/impala/sql/load_tpcds_parquet.sql
          ${IMPALA_SHELL} -q "select count(*) from tpcds_parquet.store_sales;"
      volumeMounts:
        - name: warehouse
          mountPath: /user/hive/warehouse
  volumes:
    - name: warehouse
      persistentVolumeClaim:
        claimName: impala-impala-warehouse
YAML
EOF
```

### 4b.2 Watch progress
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'kubectl -n impala logs -f tpcds-loader'
```

### 4b.3 Run queries after load
Use the Parquet database name `tpcds_parquet`:
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'kubectl -n impala run tpcds-query \
   --image=apache/impala:4.5.0-impala_quickstart_client \
   --restart=Never --command -- \
   impala-shell --protocol=beeswax \
   --impalad=impala-impala-impalad.impala.svc.cluster.local:21000 \
   --query "select count(*) from tpcds_parquet.store_sales;"'
```

### 4b.4 Clean up pods
```bash
ssh -i /Users/anubhav/.ssh/${KEY_NAME}.pem ubuntu@${PUBLIC_IP} \
  'kubectl -n impala delete pod tpcds-loader tpcds-query --ignore-not-found'
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

## Optional: Enable TLS + LDAP (Authentication)

This minimal setup is **no-auth** by default. To enable TLS + LDAP:

### Option A: Use the built-in OpenLDAP container (dev/test)
This chart can deploy a local OpenLDAP server for development use. Enable it
and wire Impala to it with:

```bash
cat <<'EOF' > /tmp/impala-auth-values.yaml
ldapServer:
  enabled: true
  domain: "example.com"
  organisation: "Impala"
  adminPassword: "admin"
  configPassword: "config"
  tls:
    enabled: true
    secretName: impala-ldap-tls
  seedUser:
    uid: impalauser
    password: impala123
    givenName: Impala
    sn: User

auth:
  ldap:
    enabled: true
    uri: "ldaps://impala-impala-ldap.impala.svc.cluster.local:636"
    baseDN: "dc=example,dc=com"
    userSearchBaseDN: "ou=People,dc=example,dc=com"
    bindDN: "cn=admin,dc=example,dc=com"
    bindPasswordSecretCreate: true
    bindPassword: "admin"
    userFilter: "(uid={0})"
    searchBindAuthentication: true
    tlsEnabled: true
    caSecretName: impala-ldap-tls
    caFile: ca.crt
EOF
```

Create the LDAP TLS secret (example uses a self-signed cert):
```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout /tmp/ldap.key -out /tmp/ldap.crt \
  -subj "/CN=impala-impala-ldap.impala.svc.cluster.local"

kubectl -n impala create secret generic impala-ldap-tls \
  --from-file=tls.crt=/tmp/ldap.crt \
  --from-file=tls.key=/tmp/ldap.key \
  --from-file=ca.crt=/tmp/ldap.crt
```

Then upgrade:
```bash
helm upgrade impala /home/ubuntu/impala-helm -n impala \
  --set image.prefix="apache/impala:4.5.0-" \
  -f /tmp/impala-auth-values.yaml
```

Connect with LDAP:
```bash
impala-shell --ldap \
  --user impalauser \
  --ldap_password_cmd='echo impala123' \
  -i localhost:21000
```
Note: this dev OpenLDAP example uses LDAPS with a self-signed cert. For
production, replace with a real CA and rotate secrets appropriately.

#### Keep LDAP users across restarts (recommended)
OpenLDAP data is **ephemeral** unless you enable persistence. Turn it on so
users are not wiped on restart:

```yaml
ldapServer:
  persistence:
    enabled: true
    size: 2Gi
    accessModes:
      - ReadWriteOnce
    storageClassName: ""
```

Apply:
```bash
helm upgrade impala /home/ubuntu/impala-helm -n impala \
  --set image.prefix="apache/impala:4.5.0-" \
  -f /tmp/impala-auth-values.yaml
```

### Option B: Use an existing LDAP server
If you already have LDAP/AD, follow the steps below.

### 1) Create TLS secret (on EC2)
```bash
# Replace with your certs
kubectl -n impala create secret tls impala-tls \
  --cert=/path/to/tls.crt \
  --key=/path/to/tls.key

# If you have a CA cert for client verification, add it:
kubectl -n impala create secret generic impala-tls \
  --from-file=ca.crt=/path/to/ca.crt \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 2) Create LDAP bind password secret (on EC2)
```bash
kubectl -n impala create secret generic impala-ldap-bind \
  --from-literal=bind_password='REPLACE_ME'
```

### 3) Update Helm values
```bash
cat <<'EOF' > /tmp/impala-auth-values.yaml
auth:
  tls:
    enabled: true
    secretName: impala-tls
  ldap:
    enabled: true
    uri: "ldaps://ldap.example.com:636"
    baseDN: "dc=example,dc=com"
    bindDN: "cn=binduser,dc=example,dc=com"
    bindPasswordSecret: impala-ldap-bind
    bindPasswordKey: bind_password
    userFilter: "(uid={0})"
    groupFilter: ""
    searchBindAuthentication: true
EOF
```

### 4) Upgrade Impala
```bash
helm upgrade impala /home/ubuntu/impala-helm -n impala \
  --set image.prefix="apache/impala:4.5.0-" \
  -f /tmp/impala-auth-values.yaml
```

### 5) Connect with TLS + LDAP
```bash
# Example with LDAP user/password (client side)
impala-shell --ssl \
  --ldap --user your_user --ldap_password_cmd='echo your_password' \
  -i localhost:21000
```

#### TLS hostname mismatch from localhost
If your local `impala-shell` does not support `--ssl_host`, add a host alias
and connect via the service name:

```bash
sudo sh -c 'echo "127.0.0.1 impala-impala-impalad" >> /etc/hosts'

impala-shell --ssl --ca_cert=~/Downloads/impala-ca.crt \
  --ldap --user impalauser \
  --ldap_password_cmd='echo -n impala123' \
  -i impala-impala-impalad:21000 --protocol=beeswax
```

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
