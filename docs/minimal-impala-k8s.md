# Minimal Impala on Kubernetes (Bare Metal/Any VM) (End-to-End)

This guide uses an x86_64 host, starts a local k3d Kubernetes cluster,
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

- SSH client
- This repo checked out at `/Users/anubhav/Desktop/impala`
- Optional: `impala-shell` on your laptop for direct querying

## Conventions

Set these once and reuse throughout:
```bash
export HOST_IP="YOUR_HOST_OR_BARE_METAL_IP"
export SSH_USER="your_ssh_user"
export SSH_KEY="/Users/anubhav/.ssh/your-key.pem"
```

## Step 1: Prepare host (x86_64 with AVX)

> Impala images require x86_64 with AVX support. Apple Silicon local clusters
> will not run these images.

Install the required tools on the host (skip anything already installed):
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} <<'EOF'
set -euxo pipefail
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https git docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker ${USER}

# kubectl
sudo curl -sL "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
  -o /usr/local/bin/kubectl
sudo chmod +x /usr/local/bin/kubectl

# helm
curl -sL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# k3d (optional; only if you want a local single-node cluster on this host)
curl -sL https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
EOF
```

## Step 2: Create k3d cluster on host (optional)

```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'sudo k3d cluster create impala-local --servers 1 --agents 0'
```

If you already have a Kubernetes cluster, skip this step and ensure
`kubectl` and `helm` are configured to point at that cluster.

Set kubeconfig for the `${SSH_USER}` user:
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'mkdir -p ~/.kube && sudo k3d kubeconfig get impala-local > ~/.kube/config && chmod 600 ~/.kube/config'
```

## Step 3: Deploy Impala (Helm)

Copy the chart to the host:
```bash
scp -i ${SSH_KEY} -r \
  /Users/anubhav/Desktop/impala/helm/impala \
  ${SSH_USER}@${HOST_IP}:/home/${SSH_USER}/impala-helm
```

Install the release (uses Apache prebuilt images):
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'helm upgrade --install impala /home/${SSH_USER}/impala-helm -n impala --create-namespace \
   --set image.prefix="apache/impala:4.5.0-"'
```

Verify:
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'kubectl -n impala get pods -o wide'
```

## Step 4: Run a Sample Query (on host)

The easiest way is to run a short-lived `impala-shell` pod using the
quickstart client image:

```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'kubectl -n impala run impala-shell \
   --image=apache/impala:4.5.0-impala_quickstart_client \
   --restart=Never --command -- \
   impala-shell --protocol=beeswax \
   --impalad=impala-impala-impalad.impala.svc.cluster.local:21000 \
   --query "select 1"'
```

View output:
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'kubectl -n impala logs impala-shell'
```

Clean up the pod:
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'kubectl -n impala delete pod impala-shell'
```

## Step 4b: Run 1G TPC-DS Load and Queries

This runs the 1 GB (SF=1) TPC-DS data generator, loads Parquet tables, and
executes a smoke query. It uses the quickstart client image to avoid local
dependencies.

### 4b.1 Create the loader pod (generates + loads)
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} <<'EOF'
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
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'kubectl -n impala logs -f tpcds-loader'
```

### 4b.3 Run queries after load
Use the Parquet database name `tpcds_parquet`:
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'kubectl -n impala run tpcds-query \
   --image=apache/impala:4.5.0-impala_quickstart_client \
   --restart=Never --command -- \
   impala-shell --protocol=beeswax \
   --impalad=impala-impala-impalad.impala.svc.cluster.local:21000 \
   --query "select count(*) from tpcds_parquet.store_sales;"'
```

### 4b.4 Clean up pods
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'kubectl -n impala delete pod tpcds-loader tpcds-query --ignore-not-found'
```

## Step 4c: Enable Kudu (optional)

This deploys a minimal Kudu master + tserver inside the same cluster and
connects Impala to it.

### 4c.1 Enable Kudu via Helm
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} <<'EOF'
cat <<'YAML' > /tmp/kudu-values.yaml
kudu:
  enabled: true
  master:
    replicas: 1
    extraArgs:
      - --default_num_replicas=1
      - --rpc_authentication=disabled
      - --rpc_encryption=disabled
  tserver:
    replicas: 1
    extraArgs:
      - --rpc_authentication=disabled
      - --rpc_encryption=disabled
YAML

helm upgrade --install impala /home/${SSH_USER}/impala-helm -n impala \
  --set image.prefix="apache/impala:4.5.0-" \
  -f /tmp/impala-auth-values.yaml \
  -f /tmp/kudu-values.yaml
EOF
```

### 4c.2 Verify Kudu from Impala
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'kubectl -n impala run kudu-check \
   --image=apache/impala:4.5.0-impala_quickstart_client \
   --restart=Never --command -- \
   impala-shell --protocol=beeswax \
   --impalad=impala-impala-impalad.impala.svc.cluster.local:21000 \
   --query "show tables in kudu;"'

# If TLS + LDAP are enabled, use:
# impala-shell --protocol=beeswax --ssl --ca_cert=/etc/impala/tls/ca.crt \
#   --ldap --user impalauser --ldap_password_cmd='echo -n impala123' \
#   --impalad=impala-impala-impalad.impala.svc.cluster.local:21000 \
#   --query "show tables in kudu;"

### 4c.3 Ensure `reason` exists in Parquet (needed by TPC-DS)
The quickstart loader does not create `reason`. Add it once:
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} <<'EOF'
kubectl -n impala delete pod tpcds-reason-load --ignore-not-found
cat <<'YAML' | kubectl -n impala apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tpcds-reason-load
spec:
  restartPolicy: Never
  containers:
    - name: loader
      image: apache/impala:4.5.0-impala_quickstart_client
      command: ["/bin/bash","-lc"]
      args:
        - |
          set -euo pipefail
          impala-shell --protocol=beeswax --ssl --ca_cert=/etc/impala/tls/ca.crt \
            --ldap --user impalauser --ldap_password_cmd="echo -n impala123" \
            -i impala-impala-impalad.impala.svc.cluster.local:21000 <<'SQL'
          create database if not exists tpcds_raw;
          drop table if exists tpcds_raw.reason;
          create external table tpcds_raw.reason (
            r_reason_sk int,
            r_reason_id string,
            r_reason_desc string
          )
          row format delimited fields terminated by '|'
          with serdeproperties ('field.delim'='|', 'serialization.format'='|')
          stored as textfile
          location '/user/hive/warehouse/external/tpcds_raw/reason'
          tblproperties('serialization.null.format'='');

          create database if not exists tpcds_parquet;
          drop table if exists tpcds_parquet.reason;
          create table tpcds_parquet.reason (
            r_reason_sk int,
            r_reason_id string,
            r_reason_desc string
          ) stored as parquet;
          insert into tpcds_parquet.reason select * from tpcds_raw.reason;
          compute stats tpcds_parquet.reason;
          SQL
      volumeMounts:
        - name: impala-ca
          mountPath: /etc/impala/tls
          readOnly: true
        - name: warehouse
          mountPath: /user/hive/warehouse
  volumes:
    - name: impala-ca
      secret:
        secretName: impala-tls
        items:
          - key: ca.crt
            path: ca.crt
    - name: warehouse
      persistentVolumeClaim:
        claimName: impala-impala-warehouse
YAML

kubectl -n impala wait --for=condition=Ready pod/tpcds-reason-load --timeout=120s || true
kubectl -n impala logs -f tpcds-reason-load
EOF
```

### 4c.4 Load TPC-DS data into Kudu (from Parquet)
Kudu requires primary keys and they must be the leading columns. This script
creates Kudu tables with explicit primary keys and inserts data from Parquet.
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} <<'EOF'
kubectl -n impala delete pod tpcds-kudu-load --ignore-not-found
cat <<'YAML' | kubectl -n impala apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tpcds-kudu-load
spec:
  restartPolicy: Never
  containers:
    - name: loader
      image: apache/impala:4.5.0-impala_quickstart_client
      command: ["/bin/bash","-lc"]
      args:
        - |
          set -euo pipefail
          impala() {
            impala-shell --protocol=beeswax --ssl --ca_cert=/etc/impala/tls/ca.crt \
              --ldap --user impalauser --ldap_password_cmd="echo -n impala123" \
              -i impala-impala-impalad.impala.svc.cluster.local:21000 "$@"
          }

          declare -A pk
          pk[call_center]=cc_call_center_sk
          pk[catalog_page]=cp_catalog_page_sk
          pk[catalog_returns]="cr_item_sk,cr_order_number"
          pk[catalog_sales]="cs_item_sk,cs_order_number"
          pk[customer]=c_customer_sk
          pk[customer_address]=ca_address_sk
          pk[customer_demographics]=cd_demo_sk
          pk[date_dim]=d_date_sk
          pk[household_demographics]=hd_demo_sk
          pk[income_band]=ib_income_band_sk
          pk[inventory]="inv_date_sk,inv_item_sk,inv_warehouse_sk"
          pk[item]=i_item_sk
          pk[promotion]=p_promo_sk
          pk[ship_mode]=sm_ship_mode_sk
          pk[store]=s_store_sk
          pk[store_returns]="sr_item_sk,sr_ticket_number"
          pk[store_sales]="ss_item_sk,ss_ticket_number"
          pk[time_dim]=t_time_sk
          pk[warehouse]=w_warehouse_sk
          pk[web_page]=wp_web_page_sk
          pk[web_returns]="wr_item_sk,wr_order_number"
          pk[web_sales]="ws_item_sk,ws_order_number"
          pk[web_site]=web_site_sk
          pk[reason]=r_reason_sk

          impala -q "create database if not exists tpcds_kudu";

          for t in $(impala -B --quiet -q "show tables in tpcds_parquet"); do
            pk_cols=${pk[$t]:-}
            if [ -z "${pk_cols}" ]; then
              echo "Missing primary key mapping for ${t}" >&2
              exit 1
            fi

            describe=$(impala -B --quiet --output_delimiter=$'\t' -q "describe tpcds_parquet.${t}" | \
              awk 'NF==0{exit} {print $1" " $2}')

            unset col_type col_order
            declare -A col_type
            declare -a col_order
            while read -r name type; do
              col_type[$name]=$type
              col_order+=("$name")
            done <<< "${describe}"

            IFS=',' read -r -a pk_list <<< "${pk_cols}"
            create_cols=""
            select_cols=""

            for pk_col in "${pk_list[@]}"; do
              if [ -z "${col_type[$pk_col]:-}" ]; then
                echo "Missing column ${pk_col} in ${t}" >&2
                exit 1
              fi
              create_cols+="${pk_col} ${col_type[$pk_col]},"
              select_cols+="${pk_col},"
            done

            for col in "${col_order[@]}"; do
              skip=0
              for pk_col in "${pk_list[@]}"; do
                if [ "${col}" = "${pk_col}" ]; then
                  skip=1
                fi
              done
              if [ ${skip} -eq 0 ]; then
                create_cols+="${col} ${col_type[$col]},"
                select_cols+="${col},"
              fi
            done

            create_cols=${create_cols%,}
            select_cols=${select_cols%,}

            impala -q "drop table if exists tpcds_kudu.${t}";
            impala -q "create table tpcds_kudu.${t} (${create_cols}, PRIMARY KEY (${pk_cols})) stored as kudu \
              tblproperties (\"kudu.num_tablet_replicas\"=\"1\")";
            impala -q "insert into tpcds_kudu.${t} (${select_cols}) select ${select_cols} from tpcds_parquet.${t}";
          done
      volumeMounts:
        - name: impala-ca
          mountPath: /etc/impala/tls
          readOnly: true
  volumes:
    - name: impala-ca
      secret:
        secretName: impala-tls
        items:
          - key: ca.crt
            path: ca.crt
YAML

kubectl -n impala wait --for=condition=Ready pod/tpcds-kudu-load --timeout=180s || true
kubectl -n impala logs -f tpcds-kudu-load
EOF
```

### 4c.5 Run TPC-DS queries against Kudu (from your laptop)
```bash
python3 - <<'PY'
from pathlib import Path
import re

repo = Path('/Users/anubhav/Desktop/impala/testdata/workloads/tpcds/queries')
out = Path('/Users/anubhav/Downloads/tpcds-kudu-queries.sql')
queries = []

files = [repo / 'count.test'] + sorted(repo.glob('tpcds-*.test'))
for path in files:
    text = path.read_text()
    blocks = re.split(r"---- QUERY:.*?\n", text)[1:]
    for block in blocks:
        parts = re.split(r"\n---- RESULTS.*\n", block, maxsplit=1)
        sql = parts[0]
        sql = "\n".join(line for line in sql.splitlines()
                        if not line.lstrip().startswith('#'))
        sql = sql.strip()
        if sql:
            queries.append((path.name, sql))

with out.open('w') as f:
    f.write("use tpcds_kudu;\n")
    f.write("set mt_dop=1;\n")
    f.write("set mem_limit=3g;\n\n")
    for name, sql in queries:
        f.write(f"-- {name}\n")
        f.write(sql.rstrip(';') + ';\n\n')

print(out, "queries:", len(queries))
PY

RESULT=~/Downloads/tpcds-kudu-results.txt
rm -f "$RESULT"
impala-shell --protocol=beeswax --ssl --ca_cert=/Users/anubhav/Downloads/impala-ca.crt \
  --ldap --user impalauser --ldap_password_cmd="echo -n impala123" \
  -i impala-impala-impalad:21000 \
  -f /Users/anubhav/Downloads/tpcds-kudu-queries.sql \
  > "$RESULT"
```

## Step 4d: Expose Impala with ingress-nginx TCP (portable)

This option works on any Kubernetes cluster (bare metal included) if you
already run `ingress-nginx`. It exposes Impala’s TCP ports via the
ingress controller using a ConfigMap mapping.

### 4d.1 Enable TCP mapping in this chart
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} <<'EOF'
cat <<'YAML' > /tmp/tcp-ingress-values.yaml
tcpIngress:
  enabled: true
  namespace: ingress-nginx
  configMapName: tcp-services
YAML

helm upgrade --install impala /home/${SSH_USER}/impala-helm -n impala \
  --set image.prefix="apache/impala:4.5.0-" \
  -f /tmp/impala-auth-values.yaml \
  -f /tmp/tcp-ingress-values.yaml
EOF
```

### 4d.2 Ensure ingress-nginx reads the TCP ConfigMap
Your ingress-nginx controller must be started with:
```
--tcp-services-configmap=ingress-nginx/tcp-services
```

### 4d.3 Ensure ingress-nginx Service exposes ports 21000/21050
Example patch (ingress-nginx namespace may differ):
```bash
kubectl -n ingress-nginx patch svc ingress-nginx-controller --type='json' -p='[
  {"op":"add","path":"/spec/ports/-","value":{"name":"impala-beeswax","port":21000,"protocol":"TCP","targetPort":21000}},
  {"op":"add","path":"/spec/ports/-","value":{"name":"impala-hs2","port":21050,"protocol":"TCP","targetPort":21050}}
]'
```

### 4d.4 Connect from your laptop (no port-forward)
```bash
# On host, expose ingress-nginx ports on the k3d loadbalancer (one-time)
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  "k3d cluster edit impala-local --port-add '21000:21000@loadbalancer' \
   --port-add '21050:21050@loadbalancer'"

# Use the host IP + 21000/21050 from your laptop.
# If TLS is enabled, add a hosts entry so the cert hostname matches:
sudo sh -c 'echo "${HOST_IP} impala-impala-impalad" >> /etc/hosts'

# Beeswax (21000) over TLS + LDAP:
impala-shell --protocol=beeswax --ssl --ca_cert=/Users/anubhav/Downloads/impala-ca.crt \
  --ldap --user impalauser --ldap_password_cmd="echo -n impala123" \
  -i impala-impala-impalad:21000

# HS2 (21050) over TLS + LDAP:
impala-shell --ssl --ca_cert=/Users/anubhav/Downloads/impala-ca.crt \
  --ldap --user impalauser --ldap_password_cmd="echo -n impala123" \
  -i impala-impala-impalad:21050
```

After that, clients can connect to `${HOST_IP}:21000` or
`${HOST_IP}:21050` without any `kubectl port-forward`. Ensure your host
firewall allows inbound TCP on `21000` and `21050`.

If you do not expose ports via `k3d cluster edit`, fall back to using
the NodePorts from `kubectl -n ingress-nginx get svc ingress-nginx-controller`.
```

## Step 5: Local `kubectl` from your laptop (SSH tunnel + kubeconfig)

### 5.1 Generate kubeconfig on host and copy to `~/Downloads`
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'sudo k3d kubeconfig get impala-local > /tmp/impala-k3d.kubeconfig'
scp -i ${SSH_KEY} \
  ${SSH_USER}@${HOST_IP}:/tmp/impala-k3d.kubeconfig \
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
  -i ${SSH_KEY} \
  ${SSH_USER}@${HOST_IP}
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
  -i ${SSH_KEY} \
  ${SSH_USER}@${HOST_IP}
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
  -i ${SSH_KEY} \
  ${SSH_USER}@${HOST_IP}
```

Then run:
```bash
export PATH="$HOME/.local/bin:$PATH"
impala-shell --protocol=beeswax -i localhost:21000 -q "select 1"
```

### Option B: Run `impala-shell` inside the cluster
If you don’t want local install, run a one-off pod:
```bash
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
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
helm upgrade impala /home/${SSH_USER}/impala-helm -n impala \
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
helm upgrade impala /home/${SSH_USER}/impala-helm -n impala \
  --set image.prefix="apache/impala:4.5.0-" \
  -f /tmp/impala-auth-values.yaml
```

### Option B: Use an existing LDAP server
If you already have LDAP/AD, follow the steps below.

### 1) Create TLS secret (on host)
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

### 2) Create LDAP bind password secret (on host)
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
helm upgrade impala /home/${SSH_USER}/impala-helm -n impala \
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
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'helm -n impala uninstall impala || true'
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'kubectl delete ns impala --ignore-not-found'
ssh -i ${SSH_KEY} ${SSH_USER}@${HOST_IP} \
  'k3d cluster delete impala-local || true'
```
